---
title: "A Mini HAL for the Compute Board, and Two Hardware Bugs an AI Helped Me Find"
date: 2026-07-05T22:47:00-07:00
draft: false
tags: ["esp32", "hal", "firmware", "platformio", "low-power", "debugging", "ai", "hardware", "iot", "open-source"]
categories: ["Projects"]
summary: "I paired with an AI coding agent to write the firmware for my compute board. It also helped me chase a flaky boot down to a strapping pin and a debounce cap that would not let go."
---

{{< figure src="featured.png" alt="compute-board 3D render, top view" href="featured.png" target="_blank" nozoom=true >}}

The compute board booted. Then it did not. Then it did again.

That flicker in the boot log is where this story starts.

I recently finished the [compute board](/blog/compute-board/), a stackable low-power ESP32 core. A bare PCB is only half the job. It still needs firmware that knows the board: which pin turns on the secondary rail, which one reads the config button, how to put everything to sleep and pull almost nothing. So I sat down to write a small hardware abstraction layer for it, and I did most of it with an AI coding agent driving the keyboard.

Two things came out of that session. A working HAL, and two hardware bugs that had nothing to do with software.

## The setup

The compute board carries an ESP32-WROOM-32E and the same two-rail power design from my LoRa nodes. One rail is always on for the MCU. The second rail, VCC_AUX, only powers up when the ESP32 is awake, so every peripheral goes fully dark in deep sleep. One GPIO enables that rail. Another reads a config button.

The agent's first useful trick: instead of me reading the schematic pin by pin, it parsed the KiCad netlist and traced every net back to the ESP32. It came back with the map. Rail enable on GPIO13. Config button on GPIO12. LED on GPIO15. I2C, SPI, the header pins, all of it. That saved me an hour of squinting at a schematic PDF.

## Bug one: the board that would not boot

With the HAL compiled and flashed, the serial console showed this on power up:

```
rst:0x1 (POWERON_RESET)
invalid header: 0xffffffff
invalid header: 0xffffffff
...
rst:0x10 (RTCWDT_RTC_RESET)
```

Seven failed reads, then the RTC watchdog steps in, then it boots. Sometimes. It was intermittent, which is the worst kind.

The config button sits on GPIO12. On the ESP32, GPIO12 is also MTDI, one of the boot strapping pins. Its level at reset picks the flash voltage: low means 3.3V, high means 1.8V. My flash is a 3.3V part. When GPIO12 reads high at reset, the chip talks to the flash at 1.8V and reads back garbage, which is the 0xffffffff in the log.

My config button network pulled GPIO12 high with a 10k resistor. That resistor was forcing the wrong flash voltage on every boot.

The fix is to let the pin sit low at reset. The ESP32 already has an internal pull-down on GPIO12 that is active during reset, so removing the external pull-up is enough to strap it right. The button still needs a pull-up to be readable, so the HAL turns on the internal pull-up at runtime, after the strap has already been sampled. One resistor off the board, and the boot log went clean.

## Bug two: the capacitor that remembered

Removing the resistor fixed cold boots. Warm resets still failed now and then. Same invalid header, same watchdog recovery.

This one took longer to see. The config button also had a debounce capacitor to ground. During normal operation the HAL holds GPIO12 up with the internal pull-up, which charges that cap to 3.3V. On a warm reset the board never loses power, so the cap is still holding its charge when the ROM samples the strap. For a few milliseconds GPIO12 reads high, the flash voltage strap goes wrong, and the boot fails until the cap slowly bleeds down through the weak internal pull-down.

The capacitor was quietly storing the exact voltage that broke the strap, then handing it back on the next reset.

Pulling the cap fixed it. The button is clean enough without it, and the HAL can debounce in software if it ever needs to. After that every boot came up POWERON, first try, no invalid headers.

There is a firmware side to this too. When the HAL enters deep sleep it now drives GPIO12 low and latches it there, so the wake reset straps correctly instead of trusting whatever the pin happened to be holding.

## The bug that was mine, not the board's

One more, because it is a good lesson. For a while the secondary rail would not turn on at all. GPIO13 measured low even though the code drove it high. GPIO13 is an RTC pad, and the raw ESP-IDF gpio calls I reached for first did not reliably take it high. The agent found it by diffing against my LoRa firmware, which drives the same pin with plain Arduino pinMode and digitalWrite and has run for a year. I switched to those, and the rail came up.

Proven code on the bench beats the cleaner API when the pad is fighting you.

## The HAL

What came out of the session is a small PlatformIO library for the board. It does four things.

- Pin definitions traced from the schematic, in one header. No magic numbers scattered across your sketch.
- Secondary rail control, `enableAuxRail()` and `disableAuxRail()`, with `begin()` bringing it up on start.
- A config button probe. `isConfigAsserted()` returns true when the button is pressed.
- Deep sleep. One call floats every registered GPIO to high impedance, holds the rail off, powers down the RTC domains, and sleeps. The RTC domains are configurable if you want to keep RTC memory across a nap.

Using it looks like this:

```cpp
#include <ComputeBoardHal.h>

static cbhal::ComputeBoardHal board;

void setup() {
  board.begin();                     // rail up, config pin ready
  board.registerPins({25, 26, 27});  // your peripheral pins

  if (board.isConfigAsserted()) {
    // button held at boot: run your config flow
    return;
  }

  // read sensors, transmit, whatever the app does...

  board.deepSleepSeconds(60);        // Hi-Z everything, rail off, sleep
}

void loop() {}
```

One design choice I like. The parts that make decisions, which pins to float, what a button reading means, how to build the sleep plan, are plain C++ with no hardware calls. They run as unit tests on a laptop, no GPIO to mock. The thin layer that actually poked the registers is the only part that needs the chip. So most of the library gets tested on every commit with no board attached.

Code is on GitHub: [compute-board-hal](https://github.com/jescarri/compute-board-hal).

## What the AI was good at, and what it was not

It was good at the mechanical work. Tracing nets, writing boilerplate, keeping the tests green, remembering which pin was which. It was quick to form a hypothesis from a boot log and quick to check it against the datasheet.

It was also confidently wrong more than once. It spent a while blaming a misbehaving LED on the 3.3V rail being too low, until I pointed out that the same LED runs fine on my LoRa board. The move that worked was treating it like a sharp junior who has read every datasheet but has never held the board. I brought the physical truth, it brought the search speed.

## Next week

The interesting number is still missing: the real deep-sleep current. On paper the HAL is doing all the right things, GPIOs floating, rail down, RTC domains off. Next week I put it on the power analyzer and measure what it actually draws asleep. Stay tuned for that one.
