---
title: "70 cm LNA (Minikits) Build Notes, Weatherproofing, and First Results"
date: 2025-08-14T23:00:00-07:00
draft: false
tags: ["UHF", "70cm", "LNA", "preamp", "Minikits", "antennas.us", "QFH", "turnstile", "station-build"]
categories: ["Radio Projects"]
summary: "Built the Minikits 70 cm preamp (≈20 dB gain). Bench results were great; rain exposed moisture and cabling issues. Here’s what worked, what didn’t, and what I’m changing."
---

I built the [Minikits 70 cm preamp](https://www.minikits.com.au/eme237) and had a great time soldering it. Assembly was straightforward, the documentation was clear and simple, and on the bench and in the field the LNA performed exactly as I hoped: about 20 dB of gain on 433 MHz.

{{< figure src="pcb_build1.jpg" alt="Assembled PCB for the Minikits 70 cm preamp" href="pcb_build1.jpg" target="_blank" nozoom=true >}}

## LNA Testing and Bench Setup

To verify performance and bandpass characteristics, I used the **Siglent SVA-1032X** spectrum analyzer in **Tracking Generator (TG)** mode.

{{< figure src="test_bench.jpg" alt="70 cm LNA measurement setup on the bench" href="test_bench.jpg" target="_blank" nozoom=true >}}

### Setup

- **Analyzer:** Siglent SVA-1032X
- **Signal chain:**
  `TG OUT → LNA IN → (−20 dB internal attenuator) → RF IN`
- **Power:** Clean DC from a bench supply
- **TG Output Level:** −20 dBm
- **Sweep Range:** 100 MHz to 1 GHz

This configuration allowed me to characterize both gain and filtering behavior.

### Filter Behavior

The wide sweep confirms excellent out-of-band rejection and a clean passband in the 70 cm region. No unexpected spurs or harmonics.

{{< figure src="bandwidth_measurement.png" alt="Bandpass response across a wide UHF sweep" href="bandwidth_measurement.png" target="_blank" nozoom=true >}}

### Gain

With normalization applied, insertion gain measured right around **20 dB at 433 MHz**, exactly in line with Minikits specs.

{{< figure src="gain.png" alt="Measured gain near 433 MHz (~20 dB)" href="gain.png" target="_blank" nozoom=true >}}


## Antenna system

For the antenna, I’m currently using an [antennas.us 70 cm / 2 m QFH kit](https://www.antennas.us/store/p/396-UC-AMSAT-KITP-2-m-/-70-cm-Passive-Amateur-Satellite-Antenna-Kit.html). I have mixed feelings about it in this application, and the "Money Back Guarantee". With the LNA in line, it works, but I suspect a DIY turnstile would perform better. Aesthetics matter at home though the QFH is tidy, non-intrusive, and keeps the neighborhood happy.

{{< figure src="antenna_install.jpg" alt="Antenna Install" href="antenna_install.jpg" target="_blank" nozoom=true >}}

You can see how the antenna + LNA performed during an International Space Station pass:

{{< youtube u3c8jUWJXho >}}


## Field test: what rain taught me

BC gave me two consecutive rain days and the LNA did not appreciate the humidity. After water worked its way in, the LNA stopped working and I pulled it down thinking the worst, burned ICs. The good news: **once it dried out, the preamp came back to life** with no permanent damage.
{{< figure src="enclosure_beta1.jpg" alt="First enclosure attempt showing moisture points" href="enclosure_beta1.jpg" target="_blank" nozoom=true >}}

{{< figure src="enclosure_2_beta1.jpg" alt="Revised enclosure details before reseal" href="enclosure_2_beta1.jpg" target="_blank" nozoom=true >}}

---

Next steps: add pigtails to the antenna and RF Out of the LNA using LMR-240, use the enclouse that comes with the Kit + Add a Faraday Cage to reduce the noise.
