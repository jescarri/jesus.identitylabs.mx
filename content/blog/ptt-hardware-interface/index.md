+++
date = '2026-11-01T09:00:00-07:00'
draft = false
title = 'Hardware for RadioCTL: the USB sound card, the audio interface, and the PTT circuit I had to build'
tags = ['Amateur Radio', 'IC-910H', 'FTDI', 'PTT', 'Electronics', 'APRS', 'RadioCTL']
categories = ['Projects', 'Ham Radio']
series = ["The RadioCTL Project"]
summary = "Choosing the USB sound card, wiring audio into the Icom IC-910H, and discovering that CI-V keying transmits a dead carrier. Why I ended up building a two-transistor PTT gate instead of keying over CAT."
+++

So far in this series I have talked about [RadioCTL's core]({{< relref "radioctl-core" >}}), the [web audio path through Dire Wolf]({{< relref "web-audio-direwolf" >}}), and [satellite tracking]({{< relref "web-satellite-tracking" >}}). All of that is software. At some point the software has to touch a real radio, and that is where this post lives: the box of parts that sits between a USB port and the back of my Icom IC-910H.

This is the part of the project I expected to be boring. Pick a sound card, solder a few wires, done. It was not boring. The radio had opinions.

## Picking the USB sound card

The IC-910H gives you audio on two rear-panel 6-pin mini-DIN sockets, one per band (MAIN is TX/RX, SUB is receive only). I wanted both bands captured independently so Dire Wolf could run one channel per band and tell me *which* band heard a packet. That rules out any mono-input dongle.

I went with the Behringer UCA202 (the UCA222 is the same hardware, different case colour). It is a USB Audio Class device, so Linux's `snd-usb-audio` just picks it up with nothing to install. The reasons it fit:

- Stereo RCA line-in at about 27 kΩ, rated up to 1.26 V rms. The radio's AF OUT is 100-300 mV rms, so each band connects directly, no resistors, with a dozen-plus dB of headroom.
- Stereo line-out that drives the TX pad cleanly.
- −77 dB crosstalk at 1 kHz, so MAIN and SUB stay independent.

The wiring is deliberately dull. MAIN pin 5 (AF OUT) goes straight to Line-In Left, SUB pin 5 to Line-In Right. Receive needs no parts at all.

Transmit is the only place a passive component shows up: Line-Out Left feeds a 10 kΩ / 1 kΩ divider (a −20.8 dB pad) into MAIN pin 1 (DATA IN). At full software level the radio sees at most about 115 mV rms, right around its nominal input and safely under the level where it starts cancelling transmission. A runaway software gain physically cannot overdrive the radio. That pad is a hardware seatbelt, and I like hardware seatbelts.

I also added a small transistor buffer per band to read each socket's squelch line into the FTDI cable's modem-status inputs, but that is a story for another day. The interesting fight was PTT.

## The plan was to key over CAT. The plan was wrong.

RadioCTL already owns the CI-V port. The obvious design was to push-to-talk over CAT: send `1C 00 01` to key, `1C 00 00` to unkey, and read `1C 00` back to confirm. No extra wires. That is exactly what I wrote down in the first revision of the interface doc, and the IC-910H's command set backed it up: the [commissioning probe]({{< relref "radioctl-core" >}}) confirmed the radio answers `1C 00` for PTT state and accepts the keying command.

Then I tested it on the air, sending APRS frames to my own HT. The log looked perfect:

```
tx: granted owner=tnc kind=tnc
tx: released ... keyed_for=0.96s
```

The radio keyed and unkeyed cleanly every time. The HT heard a carrier and nothing else. No AFSK, no tones.

I spent a while convinced it was a level problem. It was not. The TX meter showed Dire Wolf's AFSK hitting the card's output at full scale, −3 dBFS RMS. The scope confirmed about 100 mV rms arriving at DATA IN, right where the radio wants it. ALSA was at 0 dB, no attenuation. Dire Wolf's own second channel even decoded the faint leakage of my packets, so the audio was *well-formed*. Every link in the chain was proven except one: the radio's internal routing.

Here is what I eventually worked out. The IC-910H chooses its modulation source from how you keyed it:

- Key it like a front-mic event and it listens to the mic.
- Key it from the ACC SEND line and it listens to ACC audio.
- Key it by grounding the DATA socket's PTT pin and it listens to DATA IN.

CI-V PTT behaves like a mic key-up, so the radio happily transmits a carrier and completely ignores the rear-panel audio I am feeding it. APRS goes out as silence on a shared channel, which is about the rudest thing you can do on 144.390.

(I ran this past another AI assistant mid-debug, which cheerfully told me "the IC-910H does not support PTT via CI-V." That is wrong; it keys fine. It just keys the *wrong audio path*. Verifying claims against the actual radio instead of against confident prose is a recurring theme in this project.)

So CAT keying was out, not because it does not work, but because it works in a way that is useless to me. I needed to ground the radio's DATA MAIN pin 3 (PTTP) in hardware. The whole design and the evidence behind it live in a PTT-correction ADR in the repo, which superseded my original "PTT: CAT only" decision.

## Why PTT is harder than "ground a pin"

Grounding pin 3 keys the radio. An NPN transistor to ground does that. If that were the whole requirement this would be a one-transistor circuit and a short post.

The problem is *when* it must key, and, more to the point, when it must not. My FTDI cable is an FT232R, and its RTS and DTR lines are the only spare outputs I have. But those lines are not mine alone. On this host, ModemManager probes serial ports. pyserial opens a port with both lines asserted by default. So does minicom, so does Hamlib. The kernel's `ftdi_sio` driver asserts both RTS and DTR the instant *anything* opens the port, and releases both on close.

If I keyed on plain RTS, the radio would transmit the moment any of those programs so much as glanced at `/dev/ttyUSB0`. Dire Wolf's own interface guide calls plain-RTS PTT "NOT a very good approach" for exactly this reason. I needed a gate that stays off through every accident: port open, boot, reboot, crash, `kill -9`, USB unplug, and some background daemon poking the port.

## The RTS AND NOT-DTR gate

The trick is to key only on a combination that no accidental port-open ever produces: RTS asserted AND DTR released. When any program opens the port the kernel asserts *both* lines together, in a single USB request, so "both asserted" is the one state I can rely on to mean "something generic opened the port." I make that state do nothing.

{{< figure src="ptt-schematic.png" alt="The RTS-AND-NOT-DTR PTT keying gate" href="ptt-schematic.png" target="_blank" nozoom=true >}}

The gate is two transistors:

- Q1, a BC557 PNP, sits high-side and is powered from the DTR pad itself. It can only conduct when DTR is high (released) *and* RTS pulls its base low through R1 (47 kΩ). R_BE (100 kΩ) holds it off otherwise.
- Q2, a BC547 NPN, is driven by Q1's collector through R2 (10 kΩ) and does the actual work: it pulls PTTP to ground. R3 (47 kΩ) holds it off against leakage. Q2 sinks tens of milliamps, far more than the PTT line needs.

Because Q1 is fed from DTR, if the cable is unplugged or unpowered there is no supply and the gate simply cannot key. That handles the unplugged case for free.

Two small parts earn their place. There is a clamp diode, D1 (a 1N4001), across Q1's base-emitter: when RTS is high and DTR is low, Q1's base would otherwise sit 5 V above its emitter, right at the BC557's reverse limit, so D1 clamps it to about 0.6 V.

And there is an RC debounce on Q2's base, a 4.7 µF cap (C2) plus a 1 nF ceramic for RF immunity next to a 100 W radio. The cap matters more than I expected, and it is the reason the final post exists.

The resulting truth table:

| State | RTS | DTR | Keys? |
|---|---|---|---|
| Cable unplugged / FTDI unpowered | 0 V | 0 V | no (no supply) |
| Port closed, boot, after reboot | high | high | no |
| Any program opens the port | low | low | no |
| RadioCTL keys | low | high | YES |
| RadioCTL crashes or is killed while keyed | high | high | no, unkeys at once |

That one "YES" row is a deliberate state RadioCTL sets by asserting RTS while keeping DTR released. Nothing else on the host produces it. And because `ftdi_sio` releases both lines on close, a crash or a `kill -9` drops the key immediately.

Backstopping all of it, the radio's own front-panel time-out timer is set to 3 minutes, and RadioCTL's [TX arbiter]({{< relref "radioctl-core" >}}) has its own dead-man (500 ms) and time-out (30 s for data, 180 s for voice) so a hung daemon cannot sit on the air.

Before trusting any of this I had to confirm the FT232R's EEPROM was not inverting RTS or DTR, which would flip the whole table. A read-only script in the repo dumps and decodes the EEPROM so I could check the `invert_rts` / `invert_dtr` bits without writing anything to the chip. Both were off. Good.

## The whole interface on one sheet

The PTT gate is the hard part, but it is only one corner of the board. Here is the complete schematic, with the two DATA jacks (J1 MAIN, J2 SUB), the receive lines going straight to the sound card, the R1/R2 divider that pads transmit audio into DATA IN, the per-band squelch readers (Q1 and Q2 buffering CTS and DSR), and the RTS-AND-NOT-DTR gate (Q3/Q4) down in the corner.

{{< figure src="full-schematic.png" alt="Full audio and PTT interface schematic for the Icom IC-910H: DATA MAIN and SUB jacks, the transmit audio divider, the two squelch-reader transistors, and the PTT keying gate" href="full-schematic.png" target="_blank" nozoom=true >}}

Nothing here is powered from a dedicated supply. The squelch readers borrow VCC from the FTDI cable, and the PTT gate runs entirely off the RTS and DTR lines it is reading, which is the whole reason it drops the key the instant the port closes.

## Proving the truth table

On paper the gate is airtight. On a bench, "on paper" is not a thing you get to say. Every row of that table is a claim about a real circuit, and each one has to be shown true with a scope probe on the RTS pad, the DTR pad, and Q2's collector standing in for the radio's PTT line.

That means every row, including the ones that must do nothing: unplug and replug during USB enumeration, open the port with pyserial, `cat` the port, `kill -9` the process mid-key, pull the cable while keyed. The gate has to stay quiet through every single one.

And then there is that debounce cap. A 1 ms glitch on RTS should be *rejected*, and whether it is depends entirely on C2. My first build used 1 µF and keyed on the glitch. That is a failure you only find by generating the glitch and watching the collector on a scope.

Validating a truth table is as tedious as it sounds: set up the state, trigger the scope, screenshot, write it down, next row. But a PTT circuit that keys when it should not is a transmitter interfering with a shared band, so I do it. In [the validation post]({{< relref "ptt-circuit-validation" >}}) I put the circuit on the bench, walk through the rows with scope captures, and show how one capacitor value turned a glitch from a transmission into a non-event.
