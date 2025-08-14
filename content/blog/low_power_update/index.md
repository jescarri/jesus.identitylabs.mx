---
title: "Update on the Low-Power LoRa Node"
date: 2025-08-13T23:00:00-07:00
draft: false
tags: ["LoRa", "low-power", "ESP32", "IoT", "power-optimization", "firmware"]
categories: ["Projects"]
summary: "Power draw dropped from 31 mA → 14.85 mA (active) and 703 µA → 28.2 µA (deep sleep) by raising most pull-ups to 100 kΩ and fixing GPIO states across sleep/wake."
---

Since **February 5 at 11:55 PM**, I’ve had **v1.0.0** of my custom hardware and firmware running in the field.

The nodes sample soil moisture every four hours and transmit the readings over LoRa. They’ve been steady so far.

{{< figure src="SENSOR_STATUS.png" alt="Grafana dashboard with node status" href="SENSOR_STATUS.png" target="_blank" nozoom=true >}}

I’ve been looking for more power savings, After reviewing my design notes, I tried an experiment:

I shared the schematic and firmware with ChatGPT (o3) to see what it would suggest. The feedback fell into two buckets: hardware and firmware.

## Hardware

The initial hardware suggestions weren’t usable. They proposed swapping the regulator for a same-family LDO at an **incompatible output voltage**, and adding **termination resistors or buffers** on the SPI lines to the LoRa radio. Those changes would have required a substantial board redesign.

However, the exercise did prompt a simple test on my side: I changed **all pull-up resistors (except I²C)** from **10 kΩ to 100 kΩ**. That step reduced static current noticeably. Even so, the node still drew **tens of mA in active mode** and **hundreds of µA in deep sleep**.

## Firmware

On a second pass, the analysis was more helpful. It pointed to **leakage through SPI-connected GPIOs** and recommended tightening the deep-sleep and startup sequences. I updated the firmware to:

- **Keep the V_AUX LDO enabled during deep sleep.** (This was already in place.)
- **Set all GPIOs connected to radios/sensors to high-impedance** (inputs, no pulls) **before entering deep sleep.**
- **Restore GPIO direction, pull state, and peripheral configuration on wake.**

You can see the [pull request here](https://github.com/jescarri/lora-node/pull/7)

## Results

With the pull-up changes and the revised sleep/wake handling:

- **Active (TX/RX):** from **31 mA** → **14.85 mA**
- **Deep sleep:** from **703 µA** → **28.2 µA**

## Before the changes

{{< figure src="ACTIVE_MODE_BEFORE_CHANGES.png" alt="Current consumption: active mode before changes" href="ACTIVE_MODE_BEFORE_CHANGES.png" target="_blank" nozoom=true >}}

{{< figure src="SLEEP_MODE_BEFORE_CHANGES.png" alt="Current consumption: sleep mode before changes" href="SLEEP_MODE_BEFORE_CHANGES.png" target="_blank" nozoom=true >}}

## After the changes

{{< figure src="ACTIVE_MODE_AFTER_CHANGES.png" alt="Current consumption: active mode after changes" href="ACTIVE_MODE_AFTER_CHANGES.png" target="_blank" nozoom=true >}}

{{< figure src="SLEEP_MODE_AFTER_CHANGES.jpg" alt="Current consumption: sleep mode after changes" href="SLEEP_MODE_AFTER_CHANGES.jpg" target="_blank" nozoom=true >}}

That’s a large reduction, especially in deep sleep.

Using an [IoT battery-life calculator](https://www.of-things.de/battery-life-calculator.php) with the current duty cycle, the **estimated** lifetime is about **2,970 days (~8.1 years)**. This is a model-based estimate; real-world life will depend on temperature, battery aging, and radio conditions.

{{< figure src="IOT_BATTERY_LIFE.png" alt="Battery life calculator result" href="IOT_BATTERY_LIFE.png" target="_blank" nozoom=true >}}

## What’s next

I’m letting the deployed **v1.0.0** sensors run their batteries down before I touch them. After that, I’ll swap the pull-ups on those units and update them to the latest firmware, which includes the low-power fixes and **remote firmware update** support. I’ll write a follow-up post detailing the remote update process once that rollout is complete.
