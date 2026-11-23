+++
date = '2026-11-22T09:00:00-07:00'
draft = false
title = 'Keying the IC-910 from software: the PTT gate meets radio-ctl'
tags = [ 'amateur-radio', 'golang', 'ptt', 'ic-910', 'aprs', 'ftdi', 'serial', 'direwolf' ]
categories = ['Projects']
series = ["The RadioCTL Project"]
summary = 'Wiring the bench-proven RTS-AND-NOT-DTR gate into radio-ctl, proving the real transport code on the scope, and the first APRS packet on the air (plus the three bugs that came with it).'
+++

Last time I built a two-transistor gate and proved it on the bench: the circuit grounds DATA MAIN pin 3 (PTTP) only when the FTDI cable's RTS is asserted and DTR is released. I ended [that post](/blog/ptt-circuit-validation/) by promising to wire it into the software and take it on the air. This is that post.

## Why a hardware line at all

A quick recap, since the full story is in [post 4](/blog/ptt-hardware-interface/) and [post 6](/blog/ptt-circuit-validation/). The IC-910H keys fine over CI-V: send `1C 00 01` and it transmits. The problem is that keyed that way, the radio ignores the rear DATA-socket audio and puts out a silent carrier. Grounding PTTP is what selects DATA IN as the modulation source, so for APRS and for DATA-socket voice I need the physical pin pulled low, not a CI-V command.

The gate handles that. It keys only on RTS asserted AND DTR released, which means every accidental serial-port state (port closed, port opened by some other program, process killed, cable yanked) lands on a combination that does not key. That was the whole point of the hardware. Now the software has to drive it without ever stumbling into the keyed corner by accident.

## The software design

The new work is a `--ptt rts-dtr` mode in `radio-ctl`. A few decisions shaped it.

Every path that can key the radio already went through one method, `Session.SetPTT`:

- the TNC
- browser push-to-talk
- Hamlib clients over rigctld
- the REST API

I put the hardware keyer behind that method. The TX arbiter and its interlocks needed no changes, and none of the four paths knows anything is different.

After moving RTS, radio-ctl reads the radio back with `1C 00`, a read and never a write. If the radio does not report PTT on within 250 ms, the line drops and the key fails loudly. A broken wire or a half-seated connector cannot pass as success. It is the difference between "I asserted a line and hope it worked" and "the radio says it is keyed".

There are two independent locks. The rig layer refuses PTT without `--allow-transmit`. The serial port refuses to assert RTS unless it was opened for hardware PTT, and radio-ctl only does that when `--allow-transmit` is set. Both have to agree before a line moves.

radio-ctl also owns the port. Every open does three things:

- sets `TIOCEXCL` so nobody else gets in
- enables `HUPCL` so the kernel drops the modem lines if the process dies
- releases RTS and DTR immediately, so the gate starts idle

RTS itself is toggled with `TIOCMBIS` and `TIOCMBIC`.

## Bench acceptance, radio still unplugged

Before any antenna, I ran the real transport code against the scope with the radio disconnected. The jig is the same one I used for the gate validation. A 10 kOhm resistor to +5 V stands in for the radio's PTT pull-up on DATA MAIN pin 3.

{{< figure src="bench-jig.svg" alt="Bench jig with the radio unplugged: host and FTDI cable drive RTS and DTR into the PTT gate, PTTP is pulled up to 5 V through 10 kilohms, and four scope channels probe RTS, DTR, the Q4 base and PTTP" href="bench-jig.svg" target="_blank" nozoom=true >}}

No CI-V frame is sent. The tests exercise only the transport layer, through a Go test behind the `hardware` build tag:

```sh
go test -c -tags hardware ./internal/transport
RADIOCTL_HW_PTT_DEVICE=/dev/ttyUSB0 ./transport.test
```

The test pauses at each step so a script can grab measurements, CSVs and screenshots.

The sequence test (`TestHardwarePTTSequence`) walks PTTP through eight states:

| Step | Action | PTTP | Result |
|---|---|---|---|
| 1 | Open the port, HardwarePTT off | 5.00 V | idle |
| 2 | `SetPTTLine(true)`, HardwarePTT off | 5.00 V | refused (`ErrPTTLineDisabled`) |
| 3 | Open the port, HardwarePTT on | 5.00 V | idle |
| 4 | `SetPTTLine(true)` | 0.427 V | keyed |
| 5 | `SetPTTLine(false)` | 5.00 V | unkeyed |
| 6 | `SetPTTLine(true)` | 0.427 V | keyed again |
| 7 | `Close()` while keyed | 5.00 V | unkeyed |
| 8 | Reopen in the same process | 5.00 V | idle |

At step 4 the other channels read RTS 0.45 V, DTR 5.34 V and Q4 base 1.15 V.

Step 2 matters most to me. Asking for the line without hardware PTT enabled gets a flat refusal, not a key.

{{< figure src="scope-keyed.png" alt="Four-channel scope capture of the gate in the keyed state: RTS pulled low, DTR high, Q4 base up, PTTP down near 0.43 V" href="scope-keyed.png" target="_blank" nozoom=true >}}

A note on those keyed readings. PTTP sits around 0.43 V in the keyed state rather than a dead 0 V, and so does RTS. That is a small common offset on every channel in this run, a measurement artifact, not the gate failing to pull down. The pull-down works; the whole stack of channels is shifted up by the same fraction of a volt.

The open/close test (`TestHardwarePTTOpenClose`) covers the risky moments, opening and closing the port. It runs 20 cycles of open, check-idle, close, with C4 armed on a falling edge at 2.5 V on normal trigger. If the port ever flickered through the keyed corner on its way up or down, the scope would latch that falling edge. Across all 20 cycles it never triggered, because an open only ever passes through "both asserted" and "both released", never the keyed combination.

A clean pass on that armed test is a scope that never fires, so there is no waveform to show for it. The sharper picture comes from the sequence test's own measurements. Plotting PTTP against RTS and DTR across all eight steps shows the shape of it: every open, close and reopen sits at 5 V, and PTTP only falls to 0.43 V on the two steps where radio-ctl asserts the line.

{{< figure src="ptt-sequence-states.png" alt="Measured RTS, DTR and PTTP across the eight-step open, key, unkey, close, reopen sequence: PTTP holds at 5 V for every open, close and reopen and drops to 0.43 V only on the two deliberate keys" href="ptt-sequence-states.png" target="_blank" nozoom=true >}}

The crash test (`TestHardwarePTTCrash`) keys the line (PTTP 0.427 V) and then the test binary SIGKILLs itself, exit 137. PTTP went straight back to 5.00 V as the kernel released RTS and DTR. That is `HUPCL` doing its job: if radio-ctl dies mid-transmission, the transmitter unkeys on its own.

{{< figure src="scope-crash-release.png" alt="Scope capture of the crash-release edge: PTTP snapping from about 0.43 V back to 5 V as the kernel drops the lines" href="scope-crash-release.png" target="_blank" nozoom=true >}}

## The port that would not close

The first bench run did not get as far as those clean results. It failed at step 3. After `Close()`, reopening the port returned `EBUSY`, and the process holding it busy was radio-ctl itself.

The cause is a Go gotcha worth spelling out. Calling `(*os.File).Fd()` switches the file to blocking mode. My CI-V reader goroutine was parked in a blocking `read(2)` and holding the descriptor open, so `Close()` could not close the tty until a byte arrived. With no radio attached, none did.

The tty stayed open with `TIOCEXCL` still set, which locked out the reopen. Worse, `HUPCL`, my "release the lines when we die" safety net, cannot fire on a close that never happens.

The fix routes every ioctl through `SyscallConn().Control()`, which leaves the file non-blocking so `Close()` really closes. It also clears `TIOCEXCL` with `TIOCNXCL` on the way out. `TestReopenAfterClose` covers it now.

No unit test had caught this, because a pty does not behave like a real FTDI with a reader goroutine parked on it. The scope caught it.

One exception remains on purpose. The squelch reader still uses `Fd()`, because its blocking `TIOCMIWAIT` needs blocking mode. The `TIOCNXCL` on close covers that path too.

## On the air

Same day, operator present, into a dummy load first. The command:

```
./bin/radio-ctl --device /dev/ttyUSB0 --squelch hardware --allow-transmit --ptt rts-dtr --audio-capture hw:CARD=CODEC --audio-playback hw:CARD=CODEC
```

Every grant passed the CI-V `1C 00` read-back confirm. A failed confirm denies the lease, so nothing transmits unless the radio has already told radio-ctl it is keyed.

The log of the first good transmissions:

```
18:37:21 tx: granted owner=tnc kind=tnc
18:37:22 tx: released owner=tnc reason=released keyed_for=1.12s
18:37:26 direwolf: [0.4] VA7RCV-7>APY03D,ARISS,WIDE2-1::VA7RCV   :ack8
```

That last line is my own handheld acknowledging the APRS message radio-ctl sent. The AFSK made it out of the DATA socket and over the air, which CI-V keying could never have done. A second message came back `ack9` after a 1.119 s key. Then I tried browser voice push-to-talk, which keyed and released seven times in a row, 283 to 945 ms each, all cleanly released.

{{< figure src="aprs-fi.png" alt="aprs.fi showing the VA7RCV station put on the air by radio-ctl, near Langley BC in grid CN89QC" href="aprs-fi.png" target="_blank" nozoom=true >}}

aprs.fi is the independent proof. The VA7RCV beacon radio-ctl transmitted (comment "radioctl", path `[APZRCT via ARISS,qAR,VA7GFZ-1]`, timestamp 2026-11-22 19:17:20, near Langley BC in grid CN89QC) reached the APRS-IS network through a gateway I do not control. Nothing in my own stack can fake that.

## Thirty seconds of blank carrier

An earlier attempt, at 18:31:58, had a typo in the command: `--audio-playback hw:CARD=CODE`, missing the final C. No sound card has that name, so the AFSK had nowhere to play. Direwolf's frame never drained, and the "frame finished, unkey" step never happened.

The radio stayed keyed with nothing to send. It sat there as a blank carrier until radio-ctl's 30-second transmit time-out cut it off.

The typo was not the real bug. The TX arbiter checked that a playback device was configured, but not that it was running. A bad device name should produce an error message, not half a minute of dead carrier on the air.

The arbiter now refuses to grant a transmit lease while the playback device is down, so a bad name fails before anything keys.

## Browser audio that went silent

This one was not a PTT bug, but it showed up while I was testing voice. Browser audio would sometimes be silent until I toggled "Listen" off and on. It looked tied to the microphone button, which sent me down the wrong path for a while.

I reproduced it in headless Chromium with Playwright, patching `AudioContext.createMediaStreamSource` to hang an analyser off every node the page built. Silent sessions measured -120 dB on both bands, and both bands' WebAudio nodes were being fed the same `MediaStream`.

radio-ctl sends MAIN and SUB as two WebRTC tracks under a single stream id, so the browser handed the page one stream carrying both. The UI passed that stream to each band's node, and `createMediaStreamSource` on a multi-track stream uses whichever track's id sorts first. Those ids are random UUIDs, so about half the time both "bands" played SUB, which was squelched, and the audio came up silent. Every new session re-rolled the dice, which is why switching modes seemed to cause it.

The fix was one line: build `new MediaStream([e.track])` per band so each node gets its own single-track stream. Before, 2 of 5 sessions came up silent. After, 10 of 10 had audio, with MAIN and SUB measurably independent. The cause was embarrassing and the fix was trivial, which is the usual ratio.
{{< figure src="tnc-decodes.png" alt="The radio-ctl TNC page with Direwolf running and an APRS map of decoded stations" href="tnc-decodes.png" target="_blank" nozoom=true >}}

## The decisions that paid off

Putting the hardware behind the existing `Session.SetPTT` choke point saved the most work. The arbiter, the interlocks and all four client surfaces got hardware PTT without a single change. The `1C 00` read-back was the other good call: "probably keyed" became a lease the radio itself confirms.

The two bugs that mattered came from testing against the real thing. A pty does not behave like a parked FTDI reader, so only the scope found the file-descriptor bug. And I found the silent browser audio by measuring with an analyser, after wasting time listening and guessing.

Next is [a tour of the finished web console](/blog/radio-ctl-web-console/), which ties the TNC, the audio and this PTT path together.
