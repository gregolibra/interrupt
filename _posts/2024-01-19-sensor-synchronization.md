---
title: Preventing Spurious Syncrhonization in Distributed Sensor Networks
description:
  How we prevented RF packet collisions using statistics.
author: gregory
---

<!-- excerpt start -->

Hey, I'm Gregory, I'm a lead engineer at Olibra, the company behind Bond, happy to post my first contribution here!

This article describes the problem of accidental synchronization in 
distributed sensor networks - that could lead to long periods of data loss - and how we used statistics to prevent it.

<!-- excerpt end -->

{% include newsletter.html %}

{% include toc.html %}

## A brief story
We were in the development phase of a new product for the Bond product line. The new product, at that time, was called the Bond Weather Sensor. 

![]({% img_url sensor-sync/topology.png %})

This was a compact sensor, for installation outside the user's home, for capturing weather data continuously, to automate the control of shades and louvers, preventing damage in harsh weather conditions.

The sensor contained a broad range of transducers, for measuring a set of weather parameters such as temperature, humidity, wind speed and direction, solar radiation, and rain levels.

It transmitted the data points every 10 seconds, using our Bond Sync RF protocol (2GFSK @ 434 MHz). For the majority of the acquisition period, the system was sleeping, waking up periodically to perform the measurements, and to transmit the data.

Development was going great, we had a bunch of test samples installed and working: and everything seemed good for release. In the last project release meeting, we asked ourselves:

*What would happen if the user installed two (or more) sensors and they started to transmit exactly at the same time, by chance?*

## The problem
As not exactly a *synchronization* problem (because feedback is non-existent in our scenario), if the sensors have good clock sources (i.e. lower drift) it is a possibility that, at some time in the future, the transmission period of two sensors align, as each sensor clock drifts, making them transmit at the same time.

[diagram of packets colliding]

When this occurs, as all sensors transmit with the same frequency channel, data loss is inevitable, as the simultaneous packets corrupt each other. We called this situation an *eclipse window*.

An interesting fact is that, as we increase the performance of the sensor's clocks (reducing drift, using precise crystals...) we decrease the chance of seeing the problem because the network can run for much longer without entering the eclipse window. 

However, when it occurs, the eclipse window length is proportional to the accuracy of the clocks as, when the clocks meet, they will take a much longer time to drift away. 

## Field experiments
I rushed to grab data as quickly as possible, to better understand the scenario. We had never seen the sensors losing packets and, as stated before, this could be an indication that when it happens, it would be very problematic. 

*A device for the protection of the user's home shall not fail for days by chance of its working behavior.*

Two sensors were mounted in our lab, paired to the same Bond Bridge Pro, and data was captured for analysis for many days. The following study and simulations were performed using real data captured from the sensors.

## Analysis
[diagram of the phase state]

The periodic operation of the sensors can analyzed into modular form, thus it can plotted in a closed circle, called *phase space*.

The circumference of the circle has a length equal to the period of the transmissions in seconds (it is also possible to interpret this in terms of phase angles). The dots represent an RF transmission, where the color blue is used for sensor 1 and red is used for sensor 2.

[diagram of a perfect phase space + sensors and crystals]

Imaging an ideal (and theoretical) scenario where both sensors used perfect crystals as the MCUs timebase + all possible firmware branches incurred the same amount of delay, the phase space of the system (read system as the composite Bond Bridge + Sensor 1 + Sensor 2 installation), would be static in time and a collision would never occur when the sensors start to operate with a phase misalignment greater than the length of one packet.

[diagram of the phases changing]

In the real world, the sensor's clock sources center frequencies are not equal: branching at the firmware execution level will incur delay jitter, and thermal fluctuations will make both sensors operate with different instantaneous transmission periods.

This results in a phase space that evolves in time, with the relative position of the transmissions changing continuously. Packet loss occurs when the transmission overlaps in time, considering the margins of a packet's length.

[diagram of the captured sensor data drifting]

This behavior is confirmed when looking at the 12 h data capturing interval. The trend in the time difference between transmissions becomes evident when looking into a long-term time window. 

[talks that this data was used to show the phase space]

The rate at which the time length between transmission changes is proportional to the average frequency deviation between the sensor's clocks - I'm saying here *average frequency deviation* by considering that the plotted slope contains the average of all effects described before (clock source difference, firmware jitter, thermal fluctuations). 

Regressing the data to a first-order trend, we arrive at a line equation that describes the phase drift - in the form `y = Ax + B` - where `A` is exactly the average frequency rate.

Using `A` for the captured data, it becomes possible to calculate the expected eclipse length for the sensors used in the test.

[diagram with the overlap condition]

Considering that an eclipse occurs every time the packages overlap in the air, and using the time length of the packages `L = 0.02 (20 ms)`, the expected eclipse window duration becomes `EW = A / (2 * L)`. For the data captured `EW = xxxxx` - more than xx days!

[check math]

The hypothesis having the sensors not operating properly was confirmed by real data and, probably, we had never seen this in more than four months of field tests because:

*The causes leading to the catastrophic failure were the same which made it rare.*

## A simple solution
Many solutions were investigated and prototyped. A the end the chosen one was related to the known research area of *Random Walking*.

The idea consisted of randomizing the transmission period in a binary fashion. So, before going to sleep, each sensor decides if the next transmission will occur at the expected interval `P`, or if the period will be delayed by a constant amount `P + J`. 

[animation of phase space with the jittering]

This simple modification makes a big impact on the operational phase space of the sensors. It becomes clear that the transmissions dance randomly, reducing the chance of packet overlapping as time passes.

[plot or probability over time]

If we imagine that both sensors are perfect (have the same average clock frequency) and they start transmitting at the same time, `Ts1 = 0` and `Ts2 = 0`. The first packet is certainly lost, but the advantages appear as the operation continues.

Sensor 1 will choose to delay or not the next transmission, the same happens for sensor 2. Transmission from sensor 1 will occur at `Ts1 = P or Ts1 = P + J` and for sensor 2 `Ts2 = P or Ts2 = P + J`. Four permutations are possible, with two leading to a collision and the other two not, leading to the probability of an overlap to be `1/2`.

As the operation evolves, the next collision chance is halved, as the sensors choose to delay their sleep cycle or not. The overlap probability decreases exponentially, and can be approximated by the simple equation `2 * L * (1/2)^n / P`, where `n` is the index of the transmission cycle.

It is feasible to consider that the average period of transmission will be `P' = P + J/2` and that the transmission cycle `n` is approximated by `t / P'`, leading to the collision probability being modeled by `2 * L * (1/2)^(t / P') / P'`.

## Why binary?
Many behavioral simulations were performed, with the different curves of collision probability compared. The binary distribution satisfied our expectations.

This simplification fitted well with the sensor's firmware (that was already ready for shipping) as it used a hardware time for controlling sleep, where the wakeup time could be selected in discrete steps of `250 ms`. 

A single bit in the RF payload was used to relay to the Bond Bridge the decision of the sensor - if it will or will not delay the next transmission. The single bit carrying this information and enabling the bridge for scheduling (in its radio platform) seemed elegant.

## Average probabilities
It is important to keep in mind that the expectations about the packet loss probability, which were modeled with a simple equation, talk in the sense of average probabilities.

Looking at the phase space diagram and imagining the sensors walking randomly in phase, it seems obvious that collision will exist, and it may sound like their probability will not follow the described curves.

This happens because, in practice, the collisions are being spread over time and the expected results are in terms of an average sense. The phase space continues to have a length of `P (or P')` and the package length `L` is the same. 

However, a long eclipse window has a *very low* probability of existence. As soon as the transmissions align, the trend has a `50%` chance of being broken in the next cycle, and this chance increases as the operation continues.

The new behavior is aligned with the expectations of the product, as we should be constantly monitoring the weather conditions, and long periods of non operating are unacceptable. Eventual package losses were expected by the intrinsic operation of a radio phy layer.


<!-- Interrupt Keep START -->
{% include newsletter.html %}

{% include submit-pr.html %}
<!-- Interrupt Keep END -->

{:.no_toc}

## References

<!-- prettier-ignore-start -->
- [Nordic nRF-Connect SDK](https://developer.nordicsemi.com/nRF_Connect_SDK/doc/2.5.1/nrf/index.html)
- [Zephyr Getting Started Guide](https://docs.zephyrproject.org/3.5.0/develop/getting_started/index.html)
- [GitHub Actions Quickstart](https://docs.github.com/en/actions/quickstart)
- [Zephyr Docker Image](https://github.com/zephyrproject-rtos/docker-image)
- [Embedded Containers Project](https://github.com/embeddedcontainers/ncs)
- [`dive` Docker image utility](https://github.com/wagoodman/dive)
- [`dua` disk usage analyzer](https://github.com/Byron/dua-cli)
<!-- prettier-ignore-end -->
