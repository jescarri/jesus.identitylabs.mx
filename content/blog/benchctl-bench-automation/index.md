+++
date = '2026-11-08T09:00:00-07:00'
draft = false
title = 'benchctl: A CLI to Automate My Bench Equipment'
tags = ['go', 'scpi', 'vxi-11', 'test-automation', 'cli', 'electronics', 'hexagonal-architecture']
categories = ['electronics', 'software']
series = ["The RadioCTL Project"]
summary = 'A Go CLI that drives my Siglent scope, function generator and multimeter over the network, so bench measurements become scriptable, repeatable and safe.'
+++

The RadioCTL hardware kept sending me back to the bench. Every time I touched the PTT circuit or the squelch buffer, I ended up poking at the same instruments: inject a tone or a logic level with the function generator, read the voltages off the scope, use the multimeter only to confirm a rail was energized, write the numbers down, change one thing, do it all again. It worked, but it was slow, and worse, it was not repeatable. "350 mV RMS, roughly" is not a measurement I can diff against last week.

So I built a tool for it. benchctl is a command-line program that drives my bench instruments over the network. A measurement becomes a command, and a validation run becomes a file I can commit.

## What it is

benchctl is a single static Go binary that speaks SCPI to three Siglent instruments: an SDS1104X-E oscilloscope, an SDG1032X function generator and an SDM3045X multimeter.

It talks to them over VXI-11 (ONC-RPC over TCP), the standard LAN instrument protocol. Output is JSON on stdout by default, with `--human` for tables, and diagnostics go to stderr. Exit codes are meaningful: `0` ok, `1` SCPI error, `2` usage, `3` unreachable, `4` verification failed. That last one matters, because it means a failed assertion in a plan fails the process, which means CI can gate on it.

The whole point is that an agent, a script, or a `make` target can drive the bench the same way every time, and I can read the result mechanically instead of squinting at a screen.

## Why I built it

Three reasons, in order of how much they annoyed me.

Repeatability came first. The RadioCTL validation work needed the same stimulus applied the same way, run after run, so that "the squelch buffer still clips at X" was a claim I could re-run rather than remember.

Then scriptability. Front-panel knobs do not go in git. A command does. Once the bench is a CLI, a validation procedure is just a file next to the hardware docs.

And network control. My instruments sit on a bench subnet. I wanted to drive them from my laptop, from a container, or from an agent, without being in the room. VXI-11 gives me that over plain TCP.

## The architecture

I built benchctl the way I build most things now: hexagonal-ish, so the interesting logic never imports a socket. The layers are visible right in the package layout:

```
cmd/benchctl/            CLI adapter (urfave/cli/v3), JSON/human output
internal/bench/domain/   pure types: units, measurements, inventory, error kinds
internal/bench/port/     interfaces: Transport, SCPI, Scope, Generator, DMM, ...
internal/bench/app/      use cases: discover, status, measure, capture, wiring, apply, safe
internal/bench/adapter/
  transport/vxi11/       ONC-RPC/XDR VXI-11 client (core + abort channels)
  transport/raw/         TCP 5025 with block/BMP framing
  scpi/                  session, error checkers, IEEE 488.2 blocks, file lock, trace
  siglent/               driver registry + sds1000xe, sdg1000x, sdm3000x
```

- The domain holds pure types: units, waveforms, measurements, the bench inventory, error kinds. No I/O.
- The port package is just interfaces, the contract the application depends on. `Transport` moves bytes, `SCPI` is the error-checked command layer, and the role interfaces are `Scope`, `Generator`, `DMM`, `Screenshotter` and `Outputs`.
- The app layer implements the use cases (discover, status, measure, apply, safe) against those interfaces and nothing else.

Everything that touches the real world is an adapter. The VXI-11 transport is a hand-rolled ONC-RPC/XDR client with its own core and abort channels. There is a raw TCP 5025 transport as a last resort, because the SDS1000X-E leaks one raw session per connection, so VXI-11 is the default and raw is opt-in with `--transport raw`.

The per-instrument drivers live under `siglent/`, and a small registry picks one from the `*IDN?` model string: `SDS...` is a scope, `SDG...` a generator, `SDM...` a multimeter. An unrecognized model still opens as a generic instrument with SCPI-1999 error checking, so `raw` and `script` always work even on hardware I have not written a driver for yet.

The payoff of all this: the entire app layer is tested against a fake transport, so the measurement and assertion logic runs in CI with no hardware attached. Hardware tests live behind a build tag and only run when I want to drive outputs into the scope.

## The features that earn their keep

### Discover and ping

First contact is a subnet scan that pins instruments by serial number into an inventory file at `~/.config/benchctl/bench.json`:

```sh
benchctl discover --subnet 192.168.2.0/24 --save
benchctl --human ping    # reach, serial, *IDN? round trip
benchctl --human status
```

Because instruments are pinned by serial, DHCP can hand out a new address and benchctl still knows which box is the scope.

### Human-in-the-loop wiring verify

This is my favorite, because it catches the mistake I make most: a cable in the wrong jack. benchctl cannot plug in a BNC for me, but it can tell me what to plug and then check electrically that I did it right:

```sh
benchctl wiring show   [--plan plan.yaml]   # instructions for the human
benchctl wiring verify [--plan plan.yaml]   # electrical check, outputs left OFF
```

The verify step names swaps, catches a 50 Ω/HiZ mismatch, a 10X probe switch in the wrong position, or a bad contact, and leaves every output off when it is done.

### Direct instrument control

Each instrument has first-class subcommands. Note that outputs are never enabled implicitly; you turn them on yourself:

```sh
benchctl fg set 1 --wave sine --freq 1k --amp 1Vpp --offset 0 --load hiz
benchctl fg output 1 on
benchctl scope setup 1 --vdiv 200m --coupling dc --probe 1x --on
benchctl scope trigger --source C1 --mode auto --level 0
benchctl scope measure 1 ampl rms pkpk freq
benchctl dmm config vdc && benchctl dmm read --samples 5
```

### Declarative plans with assertions

This is where benchctl stops being a remote control and becomes a test runner. A plan is a YAML file: the wiring it needs, the steps to run, and what each step should read. Here is the smoke test that ships with the repo. It runs generator CH1 into scope C1, plus an external 5 V on the multimeter:

```yaml
name: fg-scope-smoke
safe_on_exit: true          # switch every output off when the plan ends
wiring:
  - {from: "fg:CH1", to: "scope:C1"}
  - {from: "ext:5V", to: "dmm:V", expect: {function: vdc, value: 5, tolerance: "5%"}}
steps:
  - fg.set:        {ch: 1, wave: sine, freq: 1k, amp: 1Vpp, offset: 0, load: hiz}
  - scope.setup:   {ch: 1, vdiv: 200m, coupling: dc, probe: 1x, on: true}
  - scope.trigger: {source: C1, mode: auto, level: 0, slope: rise}
  - fg.output:     {ch: 1, state: on}
  - wait: 2s
  - scope.measure:
      ch: 1
      params: [ampl, rms, pkpk, freq]
      expect: {ampl: "1 ± 3%", rms: "353.6m ± 3%", freq: "1k ± 0.1%"}
  - dmm.config:    {function: vdc, range: auto}
  - dmm.read:      {samples: 5, expect: "5 ± 5%"}
```

Run it with a single command, and benchctl verifies the wiring electrically before step 1, runs the steps, checks every `expect` against the real reading, and switches the outputs off on the way out:

```sh
benchctl apply plan.yaml
```

If the RMS comes back outside `353.6m ± 3%`, the assertion fails, the process exits `4`, and I know before I have even looked at the scope. A measurement that returns `****` on the instrument becomes `null`, not a silently wrong number.

### A global safe

One command, one job: switch every output off, on every instrument that has outputs. It ignores the configured limits on purpose, so it can always kill the outputs even if the inventory is misconfigured:

```sh
benchctl safe
```

It is the first thing I reach for when something looks wrong, and it is wired in as the exit behavior of every plan via `safe_on_exit`. On a live bench, "everything off, now" should never be more than one word.

## Guardrails

A function generator driving the wrong amplitude into a circuit can cook it, so the inventory also holds limits:

- `fg.max_amp_vpp`
- `fg.max_abs_v`
- `fg.max_freq_hz`

They are checked before anything is sent by `fg set`, `fg output on`, and the `fg.set` and `fg.output` steps in a plan. An invalid limit value fails closed with a config error rather than being read as "no limit."

The escape hatches, `raw` and `script`, send SCPI exactly as written and bypass the limits by design. They are the sharp tools, for deliberate use.

## The missing half: a bench-harness

The RadioCTL measurements are no longer notes in a file. They are plans that either pass or fail.

A plan that drives three instruments is only half a validation loop, though. The other half is the thing under test: the board, its firmware, and a harness that wires them together so a code change gets measured against real signals automatically. I'm calling that the bench-harness, and I'll write it up separately.

Next up is the first real use of benchctl: putting the PTT gate on the bench and seeing whether it survives.
