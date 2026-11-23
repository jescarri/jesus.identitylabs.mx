+++
date = '2026-11-20T09:00:00-07:00'
draft = false
title = 'The Web Interface, Part 2: Satellite Tracking'
tags = ['amateur-radio', 'satellites', 'sgp4', 'doppler', 'go', 'svelte', 'orbital-mechanics']
categories = ['projects']
series = ['The RadioCTL Project']
summary = 'How RadioCTL turns a pair of TLE lines into a live sky plot and a Doppler-corrected IC-910: TLE ingestion, SGP4 propagation, pass prediction, and uplink/downlink correction that follows the dial.'
+++

This is post three in the RadioCTL series. In [the core post]({{< relref "radioctl-core" >}}) I built the radio-control layer that talks CI-V to an Icom IC-910 and keeps a clean picture of the radio's state. In [part one of the web interface]({{< relref "web-audio-direwolf" >}}) I wired audio and Direwolf into the browser. Now comes the part that got me into all of this in the first place: working the amateur satellites, with the radio steering itself through the pass.

A LEO satellite is only overhead for ten or twelve minutes, and while it's there its frequency slides by several kilohertz as it comes toward you and recedes. Doing that by hand on two bands at once is a special kind of stress. I wanted to open a browser tab, pick a satellite, and watch the IC-910 do the chasing.

{{< figure src="doppler-pipeline.svg" alt="Data flow from TLE and catalog sources through SGP4 propagation and the Doppler tracker to the radio and the browser" caption="The tracking pipeline: external data is mirrored locally, propagated with SGP4, turned into Doppler corrections, and pushed to both the radio and the browser." >}}

## Where the orbits come from

You can't predict a satellite without fresh orbital elements. The standard format is the two-line element set, or TLE: two 69-character lines that encode a snapshot of the orbit at a particular epoch, from inclination and eccentricity down to the drag term. They go stale. A set a week old can put a LEO target a degree or two off, which is enough to miss a weak signal.

`internal/satdb` keeps the local picture current from two sources. Orbital elements come from [CelesTrak](https://celestrak.org): I pull the `amateur` GP group every twelve hours, and fetch anything I track that lives outside that group one catalog number at a time.

The radio side (which transmitter is a transponder, which is a beacon, the modes and passband edges) comes from the [SatNOGS DB](https://db.satnogs.org), refreshed weekly. Both get mirrored into the store, so prediction and tracking keep working with the network unplugged. That mattered more than I expected; the field sites where this radio runs don't always have a connection.

You can also paste a TLE straight into the UI. If CelesTrak is slow to list a freshly launched satellite, grabbing the elements from anywhere and dropping them into the box gets you tracking immediately.

{{< figure src="satellites.png" alt="The Satellites page: the satellite database with AO-7 selected and its transponder details" href="satellites.png" target="_blank" nozoom=true >}}

One deliberate limitation: anything with an orbital period of 225 minutes or more gets refused rather than predicted. That's the line where the simple propagator stops being accurate and you need the deep-space model. For the LEO amateur fleet it never comes up, and I would rather return an error than point the radio at the wrong sky.

## Propagating the orbit

The math that turns a TLE and a timestamp into a position lives in `internal/orbit`, built on the [`akhenakh/sgp4`](https://github.com/akhenakh/sgp4) Go library. SGP4 is the simplified perturbations model everyone uses for near-Earth objects. It goes beyond Keplerian geometry, folding in the drag and gravitational wrinkles that move a low orbit around, which is why it needs that oddly specific drag term in the TLE.

The model hands back an Earth-centered position and velocity. From there, given the ground station's latitude, longitude, and altitude, I compute the look angles: azimuth, elevation, slant range, and, importantly, the range rate, how fast the distance between you and the satellite is changing. That last number is the whole Doppler story in one float.

Two small things bit me here and are worth flagging if you build your own:

- The library in this version derives sidereal time from the raw calendar fields, so you have to feed it UTC or the entire sky rotates by your timezone offset.
- Its built-in range-rate figure is wrong (it scales the observer's velocity by the day-length ratio instead of the Earth's rotation rate), so `internal/orbit` computes range rate itself from the satellite's velocity relative to the rotating ground station, projected onto the line of sight.
## Predicting passes

A pass is one arc over your horizon: AOS (acquisition of signal, the satellite rising), TCA (time of closest approach, the elevation peak), and LOS (loss of signal, it setting). To find them, `orbit.Passes` walks forward in coarse 20-second steps watching the sign of the elevation. When elevation crosses zero it has bracketed a horizon crossing, then it bisects down to a tenth of a second for the exact AOS and LOS, and runs a golden-section search between them for the peak elevation at TCA.

A pass already in progress when you start looking is handled too: it walks backward to find the real AOS behind you.

The API caches these per satellite until the station moves or the elements change, because recomputing a full day of passes for a dozen birds on every page refresh is wasteful. The browser gets a sorted list: what's coming, when, how high, and from which bearing to which bearing across the sky.

## Doppler correction

This is where the orbit work meets the radio control from post one.

Doppler shift is the everyday effect: a frequency sounds higher when the source approaches and lower as it departs. For a satellite moving at kilometers per second it's substantial. On a 70 cm downlink the signal can swing roughly ±10 kHz across a pass. The correction is a clean ratio of range rate to the speed of light:

- Downlink (what you hear): you retune the receiver *down* as the satellite approaches, because what it transmits arrives shifted up.
- Uplink (what you transmit): the inverse, so the satellite's receiver hears the frequency it expects.

`internal/tracker` runs this loop once a second. Each tick it asks `internal/orbit` for the current look, pulls the range rate, applies the correction to the downlink and the uplink in the satellite's frame, and sends the result to the radio through the same radio-control layer and transmit interlocks from the core post.

How the two frequencies relate depends on the satellite:

- A linear transponder keeps uplink and downlink locked across the passband, inverted for inverting transponders, so you tune one dial and the other follows.
- An FM satellite just gets chased at the center.
- On a split-band pass the downlink sits on one VFO and the uplink on the other. On a same-band simplex channel the tracker drives the duplex offset instead, since there is only one receiver.

Two details I'm fond of.

First, the tracker never keys the transmitter. It only changes frequencies, and applies the transponder setup once at the start. In the daemon's read-only mode it computes and reports everything and sends nothing, so it is safe to leave running while you learn what it would do.

Second, it watches you back. Every tick it reads the frequencies out of the radio over CAT and compares them to what it last sent. If you've nudged the dial (on the front panel, in the web console, or from another program) it backs the Doppler out, works out where in the passband you meant to be, and carries on from there. On an FM channel, a change it didn't make means something else grabbed the radio, so it pauses rather than fight for control.

It also parks the radio on the AOS Doppler two minutes before the satellite rises, so you catch the first words instead of the first drift.

## What you see in the browser

The frontend is Svelte, served by `internal/api` over REST plus a WebSocket. While a session runs the tracker pushes a status snapshot every second, and the UI just renders whatever it's told.

{{< figure src="tracking.png" alt="The Tracking page: Doppler preview table, world map ground track, antenna gauge, polar sky plot and upcoming passes" href="tracking.png" target="_blank" nozoom=true >}}

The `SkyPlot` component is a polar chart: horizon on the outer ring, zenith at the center, north up. It draws the predicted path (green at AOS, red at LOS) with a dot for where the satellite is right now, so you can see at a glance whether this is a high overhead pass worth chasing or a low scrape along the horizon.

A `WorldMap` shows the ground track and footprint.

The `FreqPanel` is drawn to look like the IC-910's own display: downlink and uplink as the radio is set, Doppler-corrected, next to the satellite-frame frequencies and the live shift in hertz, with a passband tuner you can drag.

When you're not tracking, that same panel runs as a live preview for the selected transponder, so you can see the numbers before you commit the radio to anything.

Because the tracker reads the radio back and pushes a fresh status every second, the display stays accurate whether the last move came from the browser, the physical dial, or the tracker itself.

Software can steer frequencies, but it can't key the transmitter by itself. That takes a PTT interface, a bit of hardware between the computer and the radio, and it is where I finally pick up a soldering iron. It is [the next post]({{< relref "ptt-hardware-interface" >}}).
