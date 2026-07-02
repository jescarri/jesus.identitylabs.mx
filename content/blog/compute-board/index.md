---
title: "A Pluggable Compute Board for Low-Power Projects"
date: 2026-07-02T20:00:00-07:00
draft: false
tags: ["esp32", "pcb", "kicad", "low-power", "hardware", "iot", "open-source"]
categories: ["Projects"]
summary: "Two projects kept forcing me to redesign the same low-power ESP32 core from scratch. So I built a stackable compute board instead."
---

{{< figure src="featured.png" alt="compute-board 3D render, top view" href="featured.png" target="_blank" nozoom=true >}}

Two projects in a row pushed me toward the same conclusion: I needed a reusable low-power core instead of redesigning the MCU and power section every time.

## The e-ink soil moisture report

For this one I needed low power, but not LoRa. I reached for my [Lora ESP32 board](/blog/lora-esp32-board/) anyway, because it was the only low-power platform I had, but the Lora sensor did not exposed the remainging GPIOs, and ended up hand-soldering a replica of just its compute section (the ESP32 and the two LDOs) onto a protoboard. It took far longer than it should have.

Off-the-shelf boards weren't an option either. A NodeMCU or a generic ESP32 dev board will get you running in ten minutes, but none of them get anywhere near the deep-sleep efficiency my LoRa nodes have. That board [now runs 7 months on a single LiPo charge over WiFi](/blog/low_power_update/), with my latest changes in firmware and making the ESP32 pullups 10K instead of 1K, my Lora boards now run for 1 year, and I wasn't willing to trade that down for convenience.

## The Garmin Varia radar data logger

Still a protoboard PoC, but the requirements were clearer:

1. Low power, it'll sit on a pole or at an intersection logging vehicle speeds.
2. Write telemetry to an SD card.
3. Send data over multiple channels, this platform has to allow experimentation, Lora, APRS, LTE etc.
4. GPS, to tag readings with location.

Same story: I needed a low-power core with data-logging built in, plus room to bolt on whatever the current experiment needed.

## The fix: a stackable compute board

Both projects wanted the same thing underneath a low-power core I could extend, not rebuild. So I designed a base compute board with PC/104-style stacking headers, letting add-on boards plug directly on top of it.

- [compute-board](https://github.com/jescarri/compute-board) is the core: ESP32-WROOM-32E, the same two XC6220 LDOs I've been using since the LoRa board, a TPS3839 voltage supervisor, and a microSD slot.
- [baseboard](https://github.com/jescarri/baseboard) is the template add-on board, showing how to build a module that stacks on top of the compute board.

The SD card slot only sits on the not-always-on rail, and it's passive components otherwise resistors, nothing that draws current so adding it doesn't touch deep-sleep power consumption at all. The board keeps the two-rail split from the LoRa design: one rail always powers the MCU, the other only powers up when the ESP32 wakes from deep sleep, so peripherals like the SD card and whatever's stacked on top stay fully de-energized during sleep.

{{< figure src="board_top.png" alt="compute-board top render" href="board_top.png" target="_blank" nozoom=true >}}

{{< figure src="board_bottom.png" alt="compute-board bottom render" href="board_bottom.png" target="_blank" nozoom=true >}}

From here, the Garmin Varia logger becomes a baseboard with a GPS module and SD breakout, and the next low-power idea becomes another baseboard, without touching the MCU or power section again.

## A mistake: no thermal reliefs on the ground pours

I forgot to add thermal reliefs to the ground connections on this board, so every ground pad is soldered directly into the copper pour. Hand-soldering it was miserable, the pour sinks heat away from the pad faster than my iron could put it in, especially on the bigger ground pins.

The fix for the remaining boards I already have is to preheat them, then run them through my normal lead-free profile in my Controleo reflow oven.

## CI/CD and in-merge-request KiCad rendering

The other piece worth mentioning is the tooling around [compute-board](https://github.com/jescarri/compute-board). Every pull request that touches the schematic, PCB, or design rules triggers a GitHub Actions workflow that runs [KiBot](https://github.com/INTI-CMNB/KiBot) against the project. The same tool chain drives `kicad-cli` under the hood to run ERC and DRC, export schematic and per-layer PCB PDFs, render the board in 3D, and generate a STEP file.

The results get published to a `pr-<N>/<sha>` path on the repo's GitHub Pages branch, and a sticky bot comment lands on the PR with the ERC/DRC pass/fail counts and links straight to the rendered board: top, bottom, schematic, the works. No cloning the repo or opening KiCad just to see whether a change to a footprint or a trace actually looks right. Merging to `main` republishes the same outputs under a stable `main/` path, which is what feeds the shields.io ERC/DRC badges and board previews in the README.

## Special thanks to my sponsor

This PCB build was sponsored by [PCBWay](https://www.pcbway.com). If you've never used them, they handle PCB fabrication, assembly, and a growing list of other manufacturing services (CNC, 3D printing, injection molding) at prices and turnaround times that make hobby-scale hardware projects like this one actually feasible.

PCBWay is also running a month-long 12th anniversary campaign starting this July. During the campaign they're offering around $435 in coupons, and premium purple and pink solder mask free of charge. If you've been sitting on a board design, this is a good time to send it out. Details are on their [anniversary campaign page](https://www.pcbway.com/activity/anniversary12th.html).

## What's next

The compute-board and baseboard repos are up and building cleanly. Next is turning the Garmin Varia radar logger into an actual baseboard on top of this platform GPS,Lora, APRs etc, and the multi-channel telemetry it was designed for from the start. I'll follow up once that PoC comes off the protoboard.
