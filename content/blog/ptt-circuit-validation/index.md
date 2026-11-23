+++
date = '2026-11-17T09:00:00-07:00'
draft = false
title = 'Validating the PTT Circuit on the Bench'
tags = ["ptt", "oscilloscope", "siglent", "benchctl", "ic-910", "hardware", "ham-radio", "bench-validation"]
categories = ["Radio Projects"]
series = ["The RadioCTL Project"]
summary = "The payoff: I put the IC-910 PTT keying gate on the bench, drove it with benchctl, and watched it fail the glitch test. One capacitor swap later it passed. Here are the scope traces."
+++

{{< figure src="featured.png" alt="PTT gate bench validation, scope traces" href="featured.png" target="_blank" nozoom=true >}}

This is the last of the RadioCTL hardware posts, and the one I most wanted to write. Two posts ago I [designed the PTT keying gate](/blog/ptt-hardware-interface/): a two-transistor `RTS ∧ ¬DTR` logic gate that grounds the IC-910's DATA-socket PTT line only when the serial port is in exactly one state, and never on a port open, a boot, a crash or an unplug. Last post I built [benchctl](/blog/benchctl-bench-automation/), a small tool that drives my Siglent bench over SCPI so I can script the function generator and scope instead of twisting knobs.

Now I get to point the tool at the circuit and find out whether the design survives contact with a real oscilloscope. Spoiler: it did not, on the first try. That is the good part.

## The test bench

I wanted to watch the gate make its decision. The setup is four channels of scope, one function generator driving the two inputs, and a multimeter to sanity-check the supply rails:

| Role | Model | Use |
|---|---|---|
| Function gen | SDG1032X | CH1 drives RTS, CH2 drives DTR (0-5 V) |
| Oscilloscope | SDS1104X-E | C1=RTS, C2=DTR, C3=Q4 base, C4=PTTP |
| Multimeter | SDM3045X | 5 V pull-up rail sanity check |
| Bench PSU | external | +5 V through 10 kΩ to PTTP (stands in for the radio's pin-3 pull-up) |

The radio is not in the loop at all. Shorting the IC-910's PTT pin to ground keys the transmitter, so for bench work I replace the radio with a 10 kΩ pull-up to 5 V. When the gate "keys," it pulls that node down to ground; when it does not, the node sits near 5 V. That node is PTTP, and it is what channel 4 watches.

One caveat about the channels. C4 (PTTP) is the trustworthy one: a clean 0-5 V swing I can read pass/fail from directly. C3 (the Q4 base, at 200 mV/div) drifted its vertical offset between runs, so I read it only for shape, never for an absolute number. Every pass or fail call below comes from C4.

Before any of the real tests, `benchctl wiring verify` confirmed continuity between the generator and the scope and measured the 5 V rail at 4.998 V. It also flagged the expected ~147 kΩ cross-coupling between the RTS and DTR channels (the R9∥R10 bridge in the circuit), which is harmless as long as both inputs are actively driven, which they always are here.

## Test A: does it key only when it should?

First the static truth table. I drove each of the four input corners as plain DC and read PTTP. The gate is supposed to key in exactly one of them.

| RTS | DTR | PTTP | Keyed |
|---|---|---|---|
| 5 V | 5 V | 4.98 V | no |
| 5 V | 0 V | 4.98 V | no |
| 0 V | 0 V | 4.98 V | no |
| 0 V | 5 V | 0.008 V | YES |

That table is the whole safety argument. The radio keys only when RTS is low and DTR is high, the state RadioCTL asserts to transmit. Every other combination, including the "both asserted" state any program produces just by opening the port, leaves PTTP at 4.98 V. Pass, and it stayed a pass through everything below, because it is DC behaviour and nothing I changed later touched it.

## Test B: how fast does it turn on?

Static correctness is necessary but boring. The interesting behaviour is in the transients, which is exactly why I wanted a scripted scope. For turn-on I had RTS fall while DTR was held high, and measured how long PTTP took to follow.

{{< figure src="turnon.png" alt="Turn-on delay" href="turnon.png" target="_blank" nozoom=true >}}

RTS falls, and 1.1 ms later PTTP drops to ground. For keying a transmitter, a millisecond is nothing. Remember that number, though. It comes back.

## Test C: the glitch

This is the test the gate exists for. A serial line can glitch. A brief, unintended RTS-low blip must not key a 100 W transmitter onto a shared channel. So I generated a single 1 ms RTS-low pulse with DTR held high and watched whether PTTP so much as flinched.

On my first build it flinched all the way to the floor.

The debounce capacitor on the transistor base, C2, was 1 µF. With that value, the 1 ms glitch drove the base past threshold and PTTP collapsed to 0 V. The gate keyed the radio on a 1 ms glitch. That is a fail, and it is the exact failure the capacitor was supposed to prevent. Had I trusted the schematic and skipped the bench, this bug would have shown up on the air as a phantom transmission.

To make the base move less during that fixed 1 ms window, I bumped C2 to 4.7 µF and re-ran the same pulse.

{{< figure src="glitch.png" alt="1 ms glitch rejected: PTTP stays high" href="glitch.png" target="_blank" nozoom=true >}}

{{< figure src="scope-glitch.png" alt="Raw scope capture of the 1 ms glitch rejection test" href="scope-glitch.png" target="_blank" nozoom=true >}}

Now the glitch is rejected. PTTP (C4) holds flat at 4.90 V with no visible dip, and the Q4 base only bumps about 0.1 V, well short of where it would start turning the output transistor on. The radio does not key. Same stimulus, same topology, one bigger capacitor, opposite outcome.

## Test D: the release tail

The last test is the release, the TX tail: when I let RTS go back high, how long does the radio stay keyed before it lets go? PTTP returns high in 9.1 ms, up from 2.1 ms on the 1 µF build. A 9 ms tail at the end of a transmission is negligible for PTT, so that is fine on its own.

{{< figure src="tail.png" alt="Release tail: PTTP returns high in 9.1 ms" href="tail.png" target="_blank" nozoom=true >}}

The tail is also my favourite detail from the whole exercise. Here is how the timing numbers scaled when I went from 1 µF to 4.7 µF, a 4.7× change in capacitance:

| Test | 1 µF | 4.7 µF | ratio |
|---|---|---|---|
| A truth table | PASS | PASS | n/a |
| B turn-on | 0.84 ms | 1.1 ms | ~1.3× |
| C 1 ms glitch | FAIL | PASS | n/a |
| D release tail | 2.1 ms | 9.1 ms | ~4.4× |

If turn-on were a simple R·C affair, it should have scaled with the capacitor like the tail did, roughly 4.4×. It barely moved.

The reason is that turn-on is not set by R2·C2; it is limited by the current the driving transistor can push into the base node. That current is roughly fixed, so a bigger cap takes only a little longer to charge. The release, by contrast, is a passive discharge through R3, so it scales straight with C2, and the ~4.4× it grew is almost exactly the capacitor ratio.

That is more than a reassuring pass: it independently confirms that the part I soldered in is about 4.7 µF, which is the kind of cross-check I trust more than reading the number off the capacitor's body.

It also explains why Test C works at all. The glitch is rejected not because turn-on got slower (it barely did) but because the larger capacitor shrinks how far the base voltage swings during that fixed 1 ms window.

## How much margin there is

The gate keys only in the intended state and now rejects a 1 ms glitch.

The margin is not huge. The rejection threshold sits right around the turn-on time, ~1.1 ms, and the glitch I tested is 1 ms. So the 1 ms glitch passes with only about 10% margin. That is comfortable if my worst-case glitch is firmly at or under a millisecond, and thin if glitches can run longer.

More capacitance is a poor lever for buying margin here, precisely because of Test D's lesson: turn-on is current-limited, so piling on C2 lengthens the tail far faster than it moves the threshold.

If I ever need solid multi-millisecond margin, the right knob is R2: raise it to slow the base charge current directly, accept slightly less drive on the output transistor, and re-test.

For now, with the serial glitches this cable produces, 4.7 µF is the fitted value and the gate is good.

The [next post]({{< relref "radio-ctl-ptt-on-air" >}}) wires the gate into radio-ctl, checks that the CI-V read-back follows every key and unkey, and sends the first APRS packet that carries audio.
