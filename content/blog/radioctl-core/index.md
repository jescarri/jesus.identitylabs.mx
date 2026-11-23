+++
date = '2026-11-22T09:00:00-07:00'
draft = false
title = 'RadioCTL: A rigctld-Inspired Service That Controls the Radio'
tags = ['Amateur Radio', 'Golang', 'Icom', 'CI-V', 'Hamlib', 'Reverse Engineering', 'CAT']
categories = ['Projects', 'Ham Radio']
series = ["The RadioCTL Project"]
summary = 'How I built a modern control daemon for the Icom IC-910 by reverse-engineering Hamlib, reading the CI-V wire protocol byte by byte, and verifying every frame against the real radio read-only.'
+++

## Why I did this

My Icom IC-910H has been part of my station for years. It's a capable VHF/UHF all-mode radio, genuinely great for satellite work, but it's showing its age. The display is dim and tired, and the menu system has always driven me up the wall: too many button-presses to reach the things I change most, all buried behind a UX from another era.

I kept coming back to the same idea: skip the front panel entirely and drive the radio over the network, from a laptop, a phone, anything with a browser. A modern interface for a radio that never shipped with one, without replacing a radio I already like.

That became RadioCTL, a control service I wrote for the IC-910H. This is the first of eight posts about it. The rest cover the web interface (audio and Direwolf), satellite tracking, the PTT hardware, the bench tooling and validation that kept me from frying anything, the first on-air test, and a tour of the finished console. This post is about the core, the service that talks to the radio.

I wanted one small Go binary that did three things: speak the Hamlib `rigctld` protocol so my existing software (Gpredict, WSJT-X, fldigi) kept working unchanged, expose a REST/WebSocket API for a web console, and never key the transmitter unless I explicitly allowed it. That last requirement shaped the whole design.

## Why not just use Hamlib?

Hamlib's `rigctld` is the de-facto standard for amateur-radio control, and it already supports the IC-910. I rebuilt it anyway, for two reasons. First, I wanted a web console that looks and behaves like the radio's own front panel, with live push of every change, including when I turn the dial on the radio itself. That means a long-lived state engine and an event bus, not a request/response CLI shim.

Second, I don't fully trust myself (or a stray client) to not transmit into the wrong thing during development. I wanted a hard interlock baked into the service, refused in two independent layers.

The catch: if I was going to replace `rigctld`, I had to match it *exactly*. Any client that works against upstream must work against mine, byte for byte. That raised the bar from "write a rig driver" to "reverse-engineer the de-facto standard and prove I matched it."

## The architecture

The core is four layers, with one state engine in the middle and the dangerous world kept at the bottom.

{{< figure src="architecture.svg" alt="RadioCTL core architecture: rigctld and REST/WebSocket surfaces over a shared rig state engine, transport, and the serial port" href="architecture.svg" target="_blank" nozoom=true >}}

- `internal/civ` handles CI-V framing and the BCD codecs, plus the frame classifiers that decide whether a given frame is a read-only query, a PTT write, or any transmit frame at all.
- `internal/transport` is the serial link. It implements the same transaction semantics as Hamlib's `icom_one_transaction`: a reader goroutine splits the byte stream into frames, discards the bus echo (the IC-910 loops every frame back), matches replies to the in-flight command, treats the 6-byte `FB`/`FA` frames as ACK/NAK, and retries timeouts. Radio-pushed frames (you turning the dial) get routed to an async handler.
- `internal/rig` is the state engine. Every operation runs under a single operation lock, so a multi-frame sequence (select VFO, set, restore) is atomic. This is the same global mutex `rigctld` uses. It caches state, publishes events, and holds the transmit interlock.
- `internal/rigctld` and `internal/api` are just surfaces on top. The rigctld package is a port of Hamlib's own rigctld: the stream parser, the command table, the extended response protocol, the handshake.

Both surfaces funnel through the same `rig` engine, so the web console and a `rigctl -m 2` client see one coherent radio, never two fighting over the serial port.

## Reverse-engineering the CI-V protocol

CI-V is Icom's control bus. It's half-duplex serial, and the frames are simple once you see one:

```
FE FE 60 E0 <cmd> [sub] [data...] FD
```

- `FE FE` is the preamble
- `60` is the radio's CI-V address (the IC-910 default)
- `E0` is the controller
- `FD` ends the message

The radio answers with `FB` (ACK) or `FA` (NAK), or a data frame.

The framing is easy. The encodings are where it gets fiddly:

- Frequencies are little-endian packed BCD. 145.000.000 Hz becomes the five bytes `00 00 00 45 01`, each byte holding two decimal digits, least-significant first.
- Levels, tones and memory numbers, confusingly, use big-endian BCD.
- Modes are a single byte (`00` LSB, `01` USB, `03` CW, `05` FM) with an optional filter byte. A firmware quirk makes some IC-910 units swap the FM and RTTY bytes.

I took the byte values from Hamlib's own source: its IC-910 driver, the shared Icom definitions, and the frame code. They encode years of knowledge about what this radio does, and they are more reliable than guessing from the manual. I pulled out the command constants and sub-commands the IC-910 uses and wrote them into the `internal/civ` package.

## Watching the wire

Reading source tells you what Hamlib *intends* to send. To confirm what crosses the wire, you have to watch it.

The IC-910 reaches my machine through an FTDI USB-to-serial adapter showing up as `/dev/ttyUSB0`, which means every CI-V byte rides inside USB bulk transfers. On Linux you can capture those with `usbmon` and read them in Wireshark: load the `usbmon` module, point Wireshark at the USB bus the FTDI sits on, and you get a timestamped log of the bytes in both directions. With a frequency change in front of you, that `05 00 00 00 45 01` pattern stops being documentation and becomes a thing you can see happen.

That gave me confidence in the framing and echo behaviour. But the authoritative cross-check came from a different angle: I ran upstream `rigctld -m 3044` (Hamlib 4.6.2) and my own daemon side by side against identical emulated radios, and diffed every frame each one emitted. That's where the real surprises showed up:

- `set_mode` clobbers VOX delay. After every mode change, upstream sends `1A 03 <width>`. On the IC-910, `1A 03` *is* the VOX-delay parameter. Upstream was silently changing my VOX delay on every mode switch. RadioCTL selects the filter with the mode command's own filter byte and never touches it.
- `rig_open` writes to the radio. Upstream selects MAIN on open even for a read-only session. My `--read-only` mode skips that.
- One level set crashed the upstream daemon outright during the comparison.

None of that is a knock on Hamlib, which supports hundreds of radios. It's just the kind of thing you only find by comparing byte streams.

## Probing the real radio without transmitting

I still had to confirm what *my* radio answers, not what the emulator or the source implies. The risk is obvious: one stray PTT frame transmits into the antenna.

So I built `cmd/civ-guard`, a read-only CI-V firewall. It opens the real serial port, creates a pseudo-terminal, and publishes the pty as a stable symlink. Any CI-V client, Hamlib's `rigctl` or RadioCTL itself, points at the symlink instead of the real port:

```bash
civ-guard -device /dev/ttyUSB0 -baud 19200 -link /tmp/ic910-ro
rigctl -m 3044 -r /tmp/ic910-ro -s 19200 f
```

Host-to-radio frames are forwarded only when `civ.IsReadOnlyFrame` accepts them (payload-less queries). Everything else (set freq, memory writes, power, and above all any PTT write) is never sent to the radio. The guard answers it locally with the loopback echo plus a NAK, exactly as a rejected command would look on a real half-duplex bus. Every frame is logged with a direction tag, so I could diff two clients' byte-level exchanges.

With that in place I ran a full read-only probe of my IC-910H and wrote down what it supports. The radio NAKs more than you'd expect: no CAT RIT offset, no antenna select, no SWR/ALC meters, no DCS, no tuning-step readback. Those findings are now encoded in a byte-level emulator (`transport.FakeRig`), so the test suite fails the moment my code starts relying on a command this radio doesn't implement.

## The transmit interlock

The safety requirement became two independent refusals:

- In `internal/rig`, PTT, CW and VOX-on are refused unless the daemon was started with `--allow-transmit`.
- In `transport`, a `NoTransmitGuard` refuses the frame before it is written to the port. A `ReadOnlyGuard` goes further and refuses anything that isn't a pure query.

A client asking for `T 1` without the flag gets Hamlib's `RPRT -9` back and nothing hits the wire.

Defence in depth for a hobby project feels like overkill until you remember the alternative is a transmitter keyed into who-knows-what while you're debugging a parser.

## Where it stands

Most of the work was archaeology: Hamlib's source, real bytes on the USB bus, frame-by-frame diffs against upstream. The result is one binary that Gpredict and friends treat as a real `rigctld`, with an API underneath and a transmit interlock I trust.

Next is the web interface: a browser console that mirrors the radio's front panel, streams receive audio, and feeds a Direwolf TNC.
