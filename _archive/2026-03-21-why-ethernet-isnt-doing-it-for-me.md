---
layout: post
title: "Why Ethernet isn't doing it for me"
date: 2026-03-21
categories: [ethernet-to-tsn]
series: "Ethernet → TSN"
series_order: 1
description: "Piece 1 of the Ethernet → TSN arc — three reasons standard Ethernet falls apart for distributed data acquisition."
---

A rocket engine doesn't care about your measurement system. It will happily produce hundreds of sensor readings — temperatures, pressures, vibrations, flow rates — at thousands of samples per second, whether or not anything on the other end is ready to catch them. The job of a Data Acquisition System (DAQ, because aerospace runs on acronyms) is to be ready. Always. For all of them. At once.

Now here's the thing — the DAQ being built here isn't a single monolithic box with a rats nest of cables running into it. That would be too easy (and too heavy, and too fragile, and a nightmare to debug on a test stand where things are occasionally on fire). Instead, it's a distributed system: multiple small acquisition boards scattered across the test stand, each reading its own set of sensors, all connected over a network to a central machine that collects everything.

The obvious network choice? Ethernet. Gigabit, well-supported, dirt cheap, and more bandwidth than we'd ever need. At least, that's what it looked like on paper.

<center class="separator">• • •</center>

## One board, no problems

The first prototype was dead simple. One TI AM263x microcontroller, one AD7606C ADC reading 8 channels of analog data, one Ethernet cable running to a laptop. The ADC samples, the MCU packs the readings into a UDP packet with a timestamp, fires it off. On the laptop, a Python script catches the packets and plots them in real time.

![Single board topology — one AM263x with ADC connected directly to a laptop over 1 Gbps Ethernet]({{ site.baseurl }}/assets/images/ethernet-piece1/01_single_board.png)
*One board. One link. No contention. No problems.*

Data flows. Timestamps are monotonically increasing. Packets arrive in order. Life is good. There's nothing interesting happening here — and that's exactly the point. One producer, one consumer, one dedicated link. Ethernet at its most well-behaved. Like a highway with exactly one car on it — hard to have a traffic jam.

The trouble starts when you add a second car.

<center class="separator">• • •</center>

## The second board changes everything

The DAQ needs to measure sensors across the engine — not from one spot, but from multiple physical locations. So the architecture calls for multiple acquisition boards, each sampling its own set of channels, all reporting back to one machine. They can't all plug directly into the laptop (laptops, annoyingly, come with one Ethernet port). So they need a network switch.

A switch is, in the simplest terms, a device that connects multiple Ethernet devices into a network. Any device plugged into it can talk to any other device. Plug in three boards, they can all reach the laptop. Problem solved.

Except not really.

![Two acquisition boards connected through a network switch to one laptop]({{ site.baseurl }}/assets/images/ethernet-piece1/02_two_boards_switch.png)
*Two producers, one shared exit. This is where things start to break.*

The moment this topology exists, three questions appear. And none of them have good answers within standard Ethernet.

<center class="separator">• • •</center>

## Question 1: Did they sample at the same time?

This is the one that matters most, and it's the one that's easiest to get wrong.

Both boards are sampling at 1 kHz — every millisecond, read the ADC, pack a packet, send it. But "every millisecond" according to whom? Each board has its own crystal oscillator. Its own notion of time. Board 1's crystal says "1 ms has passed" and Board 2's crystal independently says "1 ms has passed" and those two moments are not the same moment. They were never the same moment. Nobody told them to be.

It gets worse. Crystals drift. A typical crystal oscillator on a dev board has an accuracy of around ±20-50 ppm (parts per million). That sounds small. It isn't. At 50 ppm, you're accumulating 50 microseconds of drift every second. Over a 10-minute engine test, that's 30 milliseconds. At 1 kHz sampling, that's 30 entire samples of misalignment. Board 1's sample number 600,000 and Board 2's sample number 600,000 are now measuring moments that are 30 ms apart in physical reality.

And there's no way to know this from the data alone. Both boards will happily report consecutive, perfectly-spaced timestamps from their own local clocks. Everything looks fine. The drift is silent.

![Two independent clocks drifting apart — after 600,000 samples, a 30ms gap exists between what each board thinks is now]({{ site.baseurl }}/assets/images/ethernet-piece1/03_clock_drift.png)
*At 50 ppm drift, sample #600,000 on each board refers to physical moments 30 ms apart.*

For a DAQ, this is catastrophic. The entire point of measuring multiple things simultaneously is the "simultaneously" part. If the chamber pressure spikes at t=4.000s on Board 1, and the vibration data from Board 2 shows a spike at t=4.000s (by its own clock), you want to correlate those events. You want to say "the pressure spike and the vibration spike happened at the same instant." But if the clocks have drifted, you might be correlating events that are milliseconds apart. For a rocket engine where things change fast, milliseconds matter.

Ethernet doesn't solve this. Ethernet moves frames from point A to point B. It has no concept of time, no mechanism for telling two devices "your clocks should agree." It's a postal service, not a metronome.

<center class="separator">• • •</center>

## Question 2: What happens inside the switch?

So both boards send their packets to the laptop. Both are going through the switch. Let's look at what actually happens in there.

A switch, internally, is simpler than most people think. Each port has an ingress side (where frames come in) and an egress side (where frames go out). When a frame arrives on any port, the switch reads the destination MAC address, figures out which port it needs to exit from, and sends it out that port.

The problem is: what happens when two frames need to exit the same port at the same time?

They can't. Ethernet is a serial protocol — one frame at a time on the wire. So the switch has a queue (a buffer) on each egress port. Frames that can't be sent immediately get queued up and sent in order. First in, first out.

![Inside a switch — two ingress ports feeding interleaved frames into a single egress queue toward the laptop]({{ site.baseurl }}/assets/images/ethernet-piece1/04_switch_internals.png)
*Two ingress ports feeding one egress queue. Frames interleave nondeterministically.*

Now let's think about this numerically. Both boards are sending at 1 kHz. That's 1000 packets per second, per board. The laptop's egress port sees 2000 packets per second arriving from two different ingress ports. At gigabit speeds, a typical ~200 byte DAQ packet takes about 1.6 microseconds to transmit. So 2000 packets per second is only about 3.2 ms of wire time out of every 1000 ms — less than 1% utilization. Bandwidth is not the problem.

The problem is *when* those packets arrive. Both boards sample at 1 kHz. Both boards have roughly similar processing times. So their packets tend to arrive at the switch at roughly the same time — within microseconds of each other. And when they do, one of them waits.

How long does it wait? Depends. Maybe 2 µs. Maybe 20 µs. Maybe more if there's other traffic. The point is: the delay is not constant. It's variable. And variable delay has a name in networking — *jitter*.

![The same packet taking three different trips through a switch — 2µs, 20µs, and 200µs — showing variable queuing delay]({{ site.baseurl }}/assets/images/ethernet-piece1/05_jitter.png)
*Same data. Same route. Different delay every time. This is jitter.*

For the DAQ, jitter means: even if both boards sampled at the exact same instant (which, as we established, they didn't — but hypothetically), the packets arrive at the laptop at different times. The laptop can't distinguish between "these were sampled at the same time but arrived at different times" and "these were sampled at different times." The temporal information is corrupted by the network itself.

<center class="separator">• • •</center>

## Question 3: Can anything cut the line?

Here's a scenario. The DAQ is running, both boards sampling away, data flowing to the laptop. Now imagine the system also needs to send a configuration command from the laptop to Board 1 — "change your sampling rate" or "switch to a different gain setting." That command goes through the same switch, same network.

Or imagine a health monitoring packet that Board 2 sends every second — CPU temperature, memory usage, error counts. Useful telemetry, but not time-critical.

Or imagine a firmware update being pushed to one of the boards while the other is still acquiring data.

All of these are frames. All of them enter the switch. All of them end up in the same egress queues. And in standard Ethernet, they're all treated equally. The switch doesn't know (and doesn't care) that the chamber pressure reading is more important than the firmware update chunk. First come, first served.

![An egress queue where a critical gain-change command is stuck behind four firmware update chunks]({{ site.baseurl }}/assets/images/ethernet-piece1/06_priority_queue.png)
*In standard Ethernet, there is no "excuse me, coming through."*

This is the priority problem. Not all data is created equal, but Ethernet treats it like it is. There's a VLAN priority field in the 802.1Q header (3 bits, 8 priority levels), but standard switches handle this with "best effort" — they'll *try* to send higher priority frames first, but there's no guarantee, no hard deadline, no "this frame MUST exit within 50 µs or the system fails."

For a DAQ on a test stand, this matters. If a pressure reading that's trending toward a redline gets delayed because it's stuck behind bulk telemetry in a queue, that's not just an inconvenience — that's a safety-relevant delay.

<center class="separator">• • •</center>

## The three problems, stated plainly

Strip away the specifics and what's left is this:

**1. No shared time.** Each device on the network has its own clock, its own notion of "now." Ethernet provides no mechanism for these clocks to agree. Two boards sampling "simultaneously" are only simultaneous by coincidence, and that coincidence degrades with every passing second.

**2. No deterministic delivery.** A frame's journey through a switch includes a variable queuing delay that depends on traffic conditions at the instant it arrives. The same frame, carrying the same data, can take 2 µs on one trip and 200 µs on the next. For a protocol, this is fine. For a measurement system, this is noise injected directly into your temporal reference.

**3. No priority enforcement.** All frames are equal citizens in a standard Ethernet queue. A critical real-time reading and a bulk data transfer sit in the same line and are served in the same order. There's no mechanism to guarantee that time-sensitive data gets through within a bounded delay.

Ethernet was built to move data reliably across a network. And it does that brilliantly — decades of engineering have made it fast, cheap, and ubiquitous. But it was never designed to answer the question "when." It's a logistics network, not a synchronization protocol. It can tell you *what* was sent and *where* it went, but the question of *when* something happened is entirely outside its vocabulary.

And for a DAQ — a system whose entire value proposition is knowing what happened, where, and *exactly when* — that's a fundamental mismatch.

<center class="separator">• • •</center>

## So now what?

The requirements haven't changed. The DAQ still needs multiple boards, still needs a network, still needs to know that when Board 1 says "t=4.000s" and Board 2 says "t=4.000s" they're talking about the same four seconds since the test started. The problem isn't Ethernet's speed or reliability. The problem is that Ethernet is temporally blind.

What's needed is a way to give the network a sense of time. A way for devices to agree on a shared clock, for the switch to guarantee delivery within known bounds, and for critical traffic to take priority over everything else. Not a replacement for Ethernet — that would throw away decades of ecosystem, tooling, and silicon. But an extension. Something that takes Ethernet's existing foundation and adds the one thing it was never designed to have: determinism.

That something exists. It's called Time-Sensitive Networking (TSN). And it changes everything.

*But that's the next post.*
