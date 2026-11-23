+++
date = '2026-11-21T09:00:00-07:00'
draft = false
title = 'The Web Interface, Part 1: Audio, Buffers, and Direwolf'
tags = ['amateur-radio', 'golang', 'webrtc', 'direwolf', 'aprs', 'audio', 'tnc', 'icom', 'pion', 'svelte']
categories = ['Projects']
series = ['The RadioCTL Project']
summary = 'How RadioCTL captures audio off the radio, feeds it to Direwolf as lossless PCM, streams both bands to the browser over WebRTC, and routes transmit audio back without ever double-keying the transmitter.'
+++

In the [previous post](/blog/radioctl-core/) I covered the core of RadioCTL: the Go daemon that owns the IC-910H's CI-V port and speaks a rigctld-compatible dialect so nothing else has to touch the serial line. That solved control. It said nothing about sound.

This post is about the audio half, which turned out to be the harder half. I want to hear the radio in a browser tab, talk back from a laptop mic, and run a software TNC for packet and APRS, all at the same time, through a single USB sound card and a single transmitter. That means one capture path has to fan out to three very different consumers, and exactly one transmit path has to arbitrate between a human voice and a modem that both want to key the radio.

{{< figure src="audio-pipeline.svg" alt="Block diagram of the RadioCTL audio pipeline: radio to audio hub, fanned out to Direwolf over UDP and to the browser over WebRTC, with the transmit arbiter feeding audio back to the radio" href="audio-pipeline.svg" target="_blank" nozoom=true >}}

## One card, one owner

The sound card is a cheap Behringer UCA20x hanging off the IC-910H's DATA jacks. The radio is stereo by design here: the left channel is the MAIN receiver, the right channel is SUB. That single fact drives everything downstream, because MAIN and SUB are different receivers listening to different things, and a listener wants to mute or pan them independently.

The first rule I settled on is that radio-ctl opens the card, and nobody else does. Two processes fighting over an ALSA device is a bad time, and PipeWire "helpfully" grabbing the card is worse. So the daemon opens `hw:CARD=CODEC` directly through a pure-Go ALSA binding (I'm avoiding cgo across the whole project, the same reason I use `modernc.org/sqlite`), at a fixed 48 kHz, S16LE, stereo, in 10 ms periods. Nothing resamples on this path. The format constants live in `internal/audio`:

```go
Rate         = 48000
Channels     = 2
PeriodFrames = Rate / 100 // one block = 10 ms
```

Every 10 ms, one interleaved stereo block comes off the card and goes into the hub.

## The hub and its ring buffers

`audio.Hub` is the fan-out point. A consumer subscribes and gets back a `Tap`, which is just a bounded channel of blocks plus a dropped-block counter. The hub pushes each captured block to every tap, and here is the part that matters for real-time audio: the hub never blocks on a slow consumer. If a tap's queue is full, the block is dropped and counted, and the tap sees it later as a gap in the block sequence number.

```go
// The hub pushes without ever blocking; when the queue is full the block is
// dropped and counted, so a slow consumer only hurts itself.
```

That is the whole philosophy of the buffering. There is no single global jitter buffer trying to smooth everything. There is a small bounded ring per consumer, and a slow consumer degrades alone instead of stalling the capture goroutine and causing an ALSA overrun that would hurt everyone. The recorder, for instance, detects a sequence gap and writes silence plus an overrun marker rather than drifting out of sync. The browser, as you'll see, gets its real jitter buffer for free somewhere else entirely.

The hub also runs a per-band software squelch. Outside satellite mode each band's browser track is gated so you don't listen to open-squelch hiss all day. But there's a hard carve-out: the audio going to the TNC is never gated. More on why in a second.

## Feeding Direwolf lossless PCM

RadioCTL doesn't implement AFSK demodulation. [Direwolf](https://github.com/wb2osz/direwolf) does that beautifully and has for years, so the design (written up in ADR-0001 in the repo) is to spawn Direwolf as a supervised child process and treat it as a soundcard modem that happens not to own a soundcard.

The trick is Direwolf's UDP audio input. Instead of `ADEVICE plughw:...`, the generated Direwolf config says:

```
ADEVICE UDP:7355 file:FILE=tx.fifo
ACHANNELS 2
```

So radio-ctl's `audio.UDPForwarder` subscribes to the hub, and for every 10 ms block it writes one datagram of raw S16LE interleaved stereo to `127.0.0.1:7355`. Direwolf reads that as two independent radio channels, channel 0 on MAIN and channel 1 on SUB, and decodes both. It's loopback-only on purpose: lossless PCM never leaves the host.

This is exactly why the TNC path can't be squelched or run through a codec. AFSK decode margin, and the FX.25 forward error correction on top of it, degrade the moment you lossy-compress or chop the audio. The browser can have its lightly-compressed, squelched, noise-reduced stream. Direwolf gets the raw detector audio, byte for byte. When Direwolf is stopped, the UDP writes just return `ECONNREFUSED` and get counted; nothing breaks.

Decoded frames come back to radio-ctl over Direwolf's AGW TCP interface, where a pure-Go client in `internal/tnc/agw` turns them into AX.25 frames, then APRS objects, then rows in SQLite and events on the bus. But frame decoding is a post for another day; this one is about the sound.

## Streaming to the browser: WebRTC, not WebSocket

The obvious thing is to shove PCM down a WebSocket and decode it with an AudioWorklet. I tried to talk myself into it and couldn't. WebSocket runs over TCP, and TCP's head-of-line blocking means a little Wi-Fi loss turns into steadily growing delay, with no jitter buffer, no concealment, no FEC. For a turn-taking repeater conversation, latency that only grows is useless.

So receive audio goes over WebRTC, using [Pion](https://github.com/pion/webrtc), which is pure Go and embeds right in the binary. No external SFU, no media server. Signaling is a single HTTP POST in the WHEP style: the browser sends an SDP offer to `POST /api/v1/audio/session`, radio-ctl answers, and media flows directly over UDP from then on. A `DELETE` hangs up.

The browser side, a Svelte front end, adds two receive-only transceivers up front:

```js
const main = pc.addTransceiver('audio', { direction: 'recvonly' });
const sub  = pc.addTransceiver('audio', { direction: 'recvonly' });
```

Two tracks, MAIN and SUB, kept separate all the way to the browser so WebAudio can give each its own gain, mute and pan. The satellite preset, for example, brings up the downlink and mutes the other band.

The codec today is plain G.711 PCMU: 8 kHz, 64 kbit/s, trivial to do in pure Go, and completely adequate for 300 to 3000 Hz radio audio on a LAN. Opus is wired behind an interface for later; PCMU shipped first because the pure-Go Opus ports were still immature when I needed something working.

And that jitter buffer I promised? It's the browser's. WebRTC's adaptive jitter buffer, concealment and all, has years of use behind it and runs on the client, so radio-ctl's server side stays as small bounded rings and Pion does the smoothing per listener.

## Transmit: one path, arbitrated

Two different things want to key the radio: your voice from the laptop mic, and Direwolf when it has a packet to send. They share one DATA IN pin and one PTT. They must never transmit at once.

The mic goes up over WebRTC as a single track, captured with `getUserMedia` (`autoGainControl: false`, because the radio has its own ALC).

PTT is deliberately momentary and un-stickable. While you hold the button or the space bar, the browser sends `{"ptt":true,"seq":n}` every 100 ms over a WebRTC data channel named `ptt`, and the server unkeys after 500 ms of silence. Close the tab, drop the Wi-Fi, suspend the laptop, and the transmitter unkeys on its own. The server's dead-man timer is the authority. The browser is only best-effort.

Direwolf's transmit audio is a separate story, and a slightly comical one. Direwolf 1.7 can only *play* into an ALSA device, so I point it at the ALSA `file` plugin, which writes raw stereo into a named FIFO in its working directory. (Direwolf truncates long device names, so it has to be a short relative path.) On the Go side, `audio.LoopbackSource` reads that FIFO back as an audio source. Each transmission arrives as a burst of samples.

Both sources converge on the TX arbiter (`internal/txarb`). There is at most one active lease, owned by either `voice:<session>` or `tnc`. The generated Direwolf config has no PTT line at all, enforced by a unit test, so Direwolf cannot key anything.

When TX audio shows up in the FIFO, radio-ctl asks the arbiter for the lease. It gets one only if all of these hold:

- transmit is allowed
- a callsign is configured
- no voice lease is active
- the frequency is in the right window

Then it keys CI-V, waits for the ACK, pauses 30 ms for switch-over, and plays the queued audio from its first sample. When the queue drains it releases the lease, after a short tail hold and a PTT read-back to confirm the radio unkeyed. Every grant, deny and release is logged and published on the bus.

The payoff is that there is exactly one place in the code where the transmitter gets keyed, for voice and data alike. I can unit-test it against a fake radio and assert the ordering, the drain-before-unkey, the watchdog timeout, and that Direwolf never found its way to a real PTT path.

## Next

The part I'd keep if I rewrote everything is the buffering: small bounded rings that fail in isolation, with the real jitter smoothing left to the browser.

Next is [satellite tracking](/blog/web-satellite-tracking/): Doppler correction across both VFOs, and hearing your own downlink while you transmit on the uplink. That full-duplex trick leans on everything here.
