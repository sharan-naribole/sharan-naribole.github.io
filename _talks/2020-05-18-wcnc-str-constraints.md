---
title:  "802.11be MLO Constraints talk at WCNC 2020"
collection: talks
type: "Talk"
permalink: /talks/wcnc-2020-str-constraints
date: 2020-05-18 12:00:00 -0600
redirect_from: 
  - /2020/05/18/wcnc-2020-str-constraints.html
---
Presentation on MLO Constraints at [IEEE WCNC 2020](https://wcnc2020.ieee-wcnc.org/program/full-program)  

## Abstract

The next-generation IEEE 802.11 standard project, IEEE 802.11be, is focused to meet the growing demands 
of applications including high throughput, low latency and high reliability. With the emergence of dual-radio 
end user devices (STAs) and tri-band Access Points (APs), efficient operation over multiple channels distributed 
over multiple bands is a key technology being discussed in IEEE 802.11be task group to achieve the desired objectives. 
Due to insufficient channel separation in frequency, STAs might be unable to perform simultaneous transmit and receive
operations over the multiple channels in an asynchronous manner. To maximize the medium utilization of such constrained 
STAs participating in asynchronous multi-channel operation, we design and analyze Constraint Aware Asynchronous multi-channel 
operation (CA-ASYNC) protocol that includes an opportunistic 802.11 backoff resumption technique applied by the constrained 
STAs and multichannel busy state indication by the AP to the constrained STAs. Our results show that (a) CA-ASYNC’s 
opportunistic backoff resumption technique significantly improves the medium utilization for constrained STAs compared 
to alternative strategies and (b) the multi-channel busy status indication significantly decreases the collisions due 
to constrained STAs and improves access delay performance.

## Slides

[Download slides][slides]

<p align = "center">
<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/wcnc_2020_str.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>
</p>


### Slide 1: IEEE 802.11be Multi-Channel Operation

IEEE 802.11be represents the next-generation Wi-Fi standard beyond the capabilities of 802.11ax products which are now being deployed. Concurrently, there has been an emergence of 802.11 devices
with multiple radios, capable of operating simultaneously on multiple channels possibly distributed over multiple bands. Being able to send data from a traffic session using the first available channel among multiple channels has potential to improve throughput and reduce latency. Therefore, this is a key feature that will be part of 802.11be standard. By default, for any device, the medium access on the multiple channels will be independent. This might lead to possible simultaneous transmission and reception in an asynchronous manner as shown below.

### Slide 2: STR Capability

For simplicity, we denote such simultaneous transmission and reception as STR. A multi-radio device may lack the STR capability due to in-device interference caused by insufficient frequency separation of the operating channels. In other words, the device lacks ability to perform reception on one channel while transmitting on the other channel. Typically, AP devices are many-antenna systems and the AP establishes the channels of operation. Therefore, it is reasonable to assume that AP maintains STR capability always. In contrast, the STAs might lack STR capability for particular set of operating channels due to smaller form factor and simpler design compared to AP. We hereby denote such STAs as non-STR STAs.

### Slide 3: Challenging Scenarios with non-STR STAs

Due to their operation constraint, non-STR STAs can degrade asynchronous operation performance at both device scale and the network scale. At the device scale, if the non-STR STA and AP transmit to each other simultaneously on both channels, it leads to reception failure at non-STR STA. At the network scale, non-STR STA can also impact other device performance. As shown in below figure, a non-STR STA cannot update its medium state information on Channel B while transmitting on Channel A. After the transmission ends on Channel A, STA 2’s transmission does not meet the energy detection threshold at non-STR STA. Consequently, non-STR STA transmits on Channel B leading to a collision at the AP. 

### Slide 4: Conservative Baseline Solution

To prevent reception failure due to STR constraint, a baseline conservative approach would be for non-STR STA to suspend transmission attempts on one channel on switching to busy state on the other channel Similarly, AP can choose not to transmit to any non-STR STA on one channel on switching to busy state on the other channel. Although this strategy prevents reception failure at non-STR STAs, it can lead to severe under-utilization of the multi-channel operation. As shown below, the frames received on Channel A may not be destined for non-STR STA yet it has deferred its medium access. Moreover, this strategy does not address the Deafness issue identified in the previous slide.

### Slide 5: Contributions

To address the new challenges introduced by non-STR STAs, we propose Constraint-Aware Asynchronous multi-channel operations or CA-ASYNC for short. To help non-STR STAs identify the scenarios they can resume backoff countdown instead of simply deferring like the baseline strategy, we propose an Opportunistic Backoff countdown resumption technique that. To address the non-STR STA deafness, we propose multi-channel busy status feedback from AP. To further boost channel access opportunity for non-STR STA, we propose protocol to aggregate 802.11 transmission opportunity over multiple channels while considering the fairness for legacy devices in the network. Due to time limit, we do not discuss this topic in this talk. We implement the key components of CA-ASYNC in a custom ns-3 simulator to analyze the performance under various network and traffic conditions.

### Slide 6: Opportunistic Backoff Countdown Resumption

We illustrate the key concept of our proposed Opportunistic technique. A non-STR STA contending on Channel A and Channel B suddenly detects a 802.11 frame on Channel A. At this point, it does not have knowledge whether this frame is from the AP with this non-STR STA as destination address or not. Therefore, it suspends backoff countdown on Channel B although that channel is idle. After decoding the PHY header of 802.11 frame on Channel A and determining this frame is indeed not for itself, it resumes backoff countdown on Channel B. To facilitate this manner of real-time countdown resumption, we use existing info in the 802.11 PHY header such as BSS Color, UL/DL flag as well as propose new information such as STA identifier for both uplink and downlink frames.

### Slide 7: Multi-Channel Busy Status Feedback

To address non-STR STA deafness issue, we utilize the knowledge at AP that the data frame received is from a non-STR STA on channel A. Accordingly, we propose indication of busy status in acknowledgement from AP to non-STR STA on channel A. Upon decoding this indication, non-STR STA can initiate a timer countdown with the timer value corresponding to the 802.11 standard maximum transmission opportunity limit. Consequently, the non-STR STA can either (a) resume contention after the countdown timer expires or (b) suspend the timer upon hearing an intra-BSS frame and update its medium access state. In this manner, the non-STR STA does not interfere with an ongoing intra-BSS transmission even if it failed to detect using existing 802.11 energy detection methods. 

### Slide 8: Performance Evaluation

We extend the ns-3 network simulator with key components of CA-ASYNC and alternative strategies. We focus on a single AP multi-channel BSS operating on two channels. We consider a varying number of non-STR STAs as well as single channel STAs and OBSS traffic flows on both channels. We focus on full-buffered uplink traffic at non-STR STAs and consider the non-STR STA uses 5.5 ms of network time when it obtains access on a channel and uses an effective data rate of 400 Mbps on each channel.

### Slide 9: CA-ASYNC’s Medium Utilization

To analyze CA-ASYNC’s medium utilization, we compare the aggregate uplink throughput of non-STR STAs. The x-axis represents the number of non-STR STAs and each sub-graph is for a different number of OBSS traffic flows. When there is a single non-STR STA and no OBSS traffic flows, all the strategies have the same throughput as non-STR STA utilizes only one channel at t a time. When there are multiple non-STR STAs and OBSS traffic flows, multi-channel baseline solution performs worse than single channel operation in some cases. This is because, in this strategy, the non-STR STAs are conservative in their medium access by deferring their contention on one channel even when a neighbor BSS transmission is occurring on the other channel. In contrast, in CA-ASYNC, non-STR STAs opportunistically resume their backoff countdown upon identifying the frame being received on other channel is either an intra-BSS uplink frame or a neighbor BSS frame.

### Slide 10: CA-ASYNC’s Busy Status Feedback

The collision at AP due to non-STR STA Deafness issue leads to increase in contention window and consequently the backoff value at the non-STR STA and the other STA whose transmission suffered. Therefore, to analyze CA-ASYNC’s busy status feedback benefit, we compare the mean backoff value with and without Busy Status Feedback for both non-STR STAs and single channel STAs. We observe a significant decrease in mean backoff value with the inclusion of Busy Status Feedback in CA-ASYNC. This is because this mechanism completely eliminates the non-STR STA Deafness issue.

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/wcnc_2020_str.pdf