---
title:  "Simultaneous NSTR Downlink talk at GLOBECOM 2020"
collection: talks
type: "Talk"
permalink: /talks/globecom-2020-nstr-dl
date: 2020-12-10 12:00:00 -0600
redirect_from: 
  - /2020/12/10/globecom-2020-simultaneous-downlink-nstr.html
---
Presentation on Simultaneous NSTR Downlink operation at [IEEE GLOBECOM 2020](https://globecom2020.ieee-globecom.org/)

## Abstract

The next-generation IEEE 802.11 standard project, IEEE 802.11be, is focused to meet the growing demands of applications including high throughput, low latency and high reliability. With the emergence of dual-radio end user devices (STAs) and tri-band Access Points (APs), efficient operation over multiple channels distributed over multiple bands is a key tech- nology being discussed in IEEE 802.11be task group to achieve the desired objectives. For certain channel combinations with insufficient frequency separation, STAs might not have the ability to simultaneously receive on one channel while transmitting on the other channel. Due to the independent medium access per-channel, there might be severe performance degradation of downlink throughput delivered to such constrained devices. To address this issue, we design and analyze Constraint-aware Aligned Downlink Ending (CADEN) protocol for simultaneous downlink transmissions over multiple channels that adaptively aligns the ending of the simultaneous downlink frames based on medium access conditions and reception capability of constrained STA. Our results show that our proposed mechanism consistently improves the downlink throughput delivered to constrained STAs under various network conditions. 

## Slides

[Download slides][slides]

<p align = "center">
<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/globecom_2020_downlink_nstr.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>
</p>


### Slide 2: 802.11be Multi-link Operation

IEEE 802.11be represents the next-generation Wi-Fi standard beyond the capabilities of 802.11ax products which are now being deployed. Concurrently, there has been an emergence of 802.11 devices with multiple radios, capable of operating simultaneously on multiple channels possibly distributed over multiple bands. Being able to send data from a traffic session using the first available channel among multiple channels has potential to improve throughput and reduce latency. 
By default, for any device, the medium access on the multiple channels will be independent. This might lead to possible simultaneous transmission and reception in an asynchronous manner. For simplicity, we denote such simultaneous transmission and reception as STR. A multi-radio device may lack the STR capability due to in-device interference caused by insufficient frequency separation of the operating channels. In other words, the device lacks ability to perform reception on one channel while transmitting on the other channel. Typically, AP devices are many-antenna systems and the AP establishes the channels of operation. Therefore, it is reasonable to assume that AP maintains STR capability always. In contrast, the STAs might lack STR capability for particular set of operating channels due to smaller form factor and simpler design compared to AP. We hereby denote such STAs as non-STR STAs.

### Slide 3: Asynchronous Operation at STR Device

The figure highlights a typical asynchronous operation at an STR capable device. The medium access on a channel is obtained independently when 802.11 backoff counter (the boxed numbers) on that channel reaches zero. The PHY data frame (PPDU) is composed of an initial PHY preamble followed by an aggregation of MAC frames or MPDUs. In response to received data, the receiver immediately  sends a block Acknowledgement comprised of a bitmap where each bit corresponds to a specific MPDU received.

### Slide 4: Simultaneous Downlink to non-STR STA

When the AP transmits simultaneously in an asynchronous manner to a non-STR STA, there is a possibility of in-device interference. As shown in the figure, the Block ACK
transmission by non-STR STA on Channel A interferes with the downlink data transmission to the same non-STR STA on channel B. Depending on the reception capability of non-
STR STA for the channels of operation, there are two extreme possibilities. At best, only the MPDU(s) being received on Channel B that overlap in time domain with Block ACK transmission on Channel A are lost as shown in this slide.

### Slide 5: Non-STR STA Receiver Capability Challenge

In contrast, in the worst case, the Block ACK transmission impacts the signal-to-noise-interference ratio on Channel B sufficient to cause the non-STR STA’s receiver on Channel B to go out of synchronization with the receiving signal and lead to failure in reception of rest of the MPDUs until the end of the data transmission as illustrated in this figure. An important point to note here is that as the AP has STR capability, it can receive the Block ACK transmission on Channel B even while transmitting to non-STR STA on Channel B.

### Slide 6: Preamble Overlap Challenge

There is another extreme case wherein start of downlink transmission including PHY preamble on Channel B overlaps with the uplink Block ACK transmission on Channel A. As the non-STR STA fails to decode PHY preamble, non-STR STA fails to receive all the MPDUs on Channel B and does not respond with Block ACK. Based on these last few slides, there might be severe downlink performance degradation if AP simply performs transmission on each channel in an asynchronous manner.

### Slide 7: Baseline Strategies

To address the reception failure at non-STR STA due to in-device interference, an AP can prevent overlap in uplink Block ACK transmission from a non-STR STA and AP’s downlink transmission to same non-STR STA. A simplistic approach would be for AP to not transmit to a non-STR STA on other channels when already involved in frame exchange with that non-STR STA. However, depending on the channel business, the medium access might be obtained by another device in the network during the AP’s deferral leading to increased delay. Another approach is for the AP to always align the ending of its simultaneous transmissions to a non-STR STA. If the non-STR STA’s reception capability is such that only the MPDUs overlapping with short Block ACK are lost, then forced alignment might lead to early ending of data transmission.

### Slide 8: Contributions

To address the new challenges introduced by non-STR STAs, we propose we propose Constraint-aware Aligned Downlink Ending or (CADEN) for short. Our proposed framework provides non-STR STAs the flexibility to indicate if they require AP to always align its downlink transmissions to the non-STR STA. At the AP side, CADEN prevents the PHY Preamble overlap with Block ACK and performs adaptive ending alignment if non-STR STA indicates alignment is not mandatory. In the next few slides, I will present more details.

### Slide 9: CADEN Procedure at non-STR STA

As mentioned previously, depending on the reception capability for the channels of operation, the MPDUs on other channel may be failed to receive even beyond the Block ACK transmission phase and in the worst case all the remaining MPDUs belonging to that transmission. We classify the potential loss into overlap loss and out-of-sync loss.
The non-STR STA indicates to AP that AP either (a) shall always align (non-STR STA can indicate this requirement if it suffers out of sync loss or if overlap loss is significant) or (b) non-STR can indicate AP may not align its simultaneous downlink data transmissions to this non-STR if non-STR STA determines the overlap loss is negligible based on channel conditions and data rate being used in the downlink.

### Slides 10 and 11: CADEN Procedures at AP

At the AP side, the adaptive behavior depends on a) non-STR STA’s reception capability b) time domain overlap of downlink data and uplink Block ACK and c) Data rate used for channel on which medium access is obtained.
As AP reserves the Channel A medium for both the data transmission and corresponding acknowledgement reception, AP has precise knowledge of start and end time of the potential Block ACK response from non-STR STA on Channel A. In addition, AP has knowledge of the start and end of PHY preamble corresponding to potential transmission on Channel B. Therefore, after 802.11 backoff counter value reaches zero for AP on Channel B, if AP determines an overlap would occur between Block ACK from a non-STR STA on Channel A and PHY preamble to same non-STR STA on Channel B, AP will not initiate data transmission over-the-air on Channel B and re-attempt transmission after the reserved medium time on Channel A by AP expires.

If the non-STR STA indicates that AP shall align the downlink transmissions, AP aligns the end of data transmission on Channel B with the end of ongoing data transmission
on Channel A. To achieve this alignment, AP might employ fragmentation and padding mechanisms already defined in existing 802.11 standard. If the non-STR STA indicates that AP may not always align the downlink transmissions, AP employs adaptive alignment procedure. AP uses the precise knowledge of
the start and end times of potential Block ACK transmission by non-STR STA on Channel A to determine the number of MPDUs that would be failed to receive by non-STR STA if the ending of data transmission on Channel B is not aligned with that on Channel A. Number of MPDUs lost depends on the data rate used on Channel B. Therefore, AP will align the ending of data transmission on both channels if the estimated MPDU loss is above a pre-defined threshold internal to the AP. Otherwise, AP will perform transmission on Channel B without any alignment with the
ongoing transmission on Channel A to the same non-STR STA.

### Slide 12: Performance Evaluation

We extend the ns-3 network simulator with key components of CADEN and alternative strategies. We focus on a single AP multi-channel BSS operating on two channels. We consider a varying number of OBSS traffic flows on both channels to inject congestion. To isolate downlink performance, we focus on full-buffered downlink traffic to a single non-STR STA. Next, we present a couple of results from the paper.

### Slide 13: Non-STR STA Reception Analysis

We validate the loss suffered by non-STR STA due to non-aligned asynchronous downlink transmissions. the x-axis represents the number of OBSS sessions on both channels in the BSS and the y-axis represents the downlink throughput delivered by AP to non-STR STA. The OBSS sessions are equally distributed over the two channels. Each sub-graph represents a different PPDU transmission time used by AP when it obtains access on the first channel to replicate varying traffic applications of different packet sizes and aggregation limit.

First, as STR STA does not suffer in-device interference, it consistently provides the highest throughput across different traffic and network conditions. Second, there is significant performance degradation for non-STR STA due to asynchronous operation as it suffers in-device interference. Third, the performance degradation is worst for non-STR STA that suffers out-of-sync loss as all the MPDUs starting from overlap with Block ACK transmission are lost. In comparison, non-STR STA that suffers overlap loss fails to receive only the MPDU(s) that overlap with the short Block ACK transmission. For shorter TXOP durations, the probability of in-device interference increases leading to preamble reception failure as well as MPDU reception failure.

### Slide 14: Comparison with Alternative Strategies

Next, we compare CADEN’s performance with alternative strategies. First, despite not performing simultaneous transmissions, as the AP still performs medium access on two channels compared to single channel operation, Defer Transmission strategy consistently betters the legacy operation. Second, however, Defer TX suffers from not utilizing the obtained medium access as OBSS traffic flows can acquire the medium instead during the deferral period. CADEN’s adaptive alignment strategy consistently provides the highest gains independent of non-STR STA’s reception capability and network conditions.


### Slide 15: Conclusion

To recap, Multi-channel operation is a key feature of next-gen IEEE 802.11be standard. In this paper, we focused on downlink data delivery to non-STR STAs. We presented CADEN, a novel design in which the non-STR STAs indicate their reception capability for simultaneous downlink transmissions and accordingly the AP adaptively constructs its data transmissions with ending alignment. In our previous work, we focused on improving uplink medium access for non-STR STAs. I also briefly highlight a few topics for future work including non-STR AP operation for example in personal hotspots and RF chain sharing across the links in a multi-channel device.

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/globecom_2020_downlink_nstr.pdf