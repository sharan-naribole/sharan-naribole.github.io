---
title:  "NSTR Mobile AP talk at ICC 2021"
collection: talks
type: "Talk"
permalink: /talks/icc-2021-nstr-map
date: 2021-06-14 12:00:00 -0600
---
Presentation for NSTR Mobile AP operation at [IEEE ICC 2021](https://icc2021.ieee-icc.org/index.html)

## Abstract

With the emergence of dual-radio end user devices (STAs) and tri-band Access Points (APs), efficient operation over multiple channels distributed over multiple bands is a key technology being discussed in the next-generation IEEE 802.11be standard task group. Recently, STAs are equipped with the functionality to temporarily operate as a mobile AP typically using cellular/WLAN backhaul. For certain channel combinations with insufficient frequency separation, STAs and therefore mobile APs might not have the ability to simultaneously receive on one channel while transmitting on the other channel. With such constraints, an asynchronous operation with independent medium access on each channel might lead to severe performance degradation. To address this issue, we design and simulate Mobile AP Medium State-based Transmission Alignment and Recess (MobiSTAR) protocol in which mobile AP opportunistically aligns the starting and ending of transmissions on multiple channels if available. Alternatively, if only a single channel is available for transmission, mobile AP aligns its transmission on that channel with ongoing out of network transmission on the other channel. Moreover, to address scenario wherein such alignment is not possible at mobile AP due to non-Wi-Fi energy received on the other channel, STAs in MobiSTAR initiate an access recess timer countdown to limit their transmission attempts. Our results show that MobiSTAR improves the downlink throughput delivered by mobile AP while minimizing the performance degradation in the uplink due to mobile AP’s simultaneous transmit-receive constraints.

## Slides

[Download slides][slides]

<p align = "center">
<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/icc_2021_mobistar.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>
</p>

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/icc_2021_mobistar.pdf