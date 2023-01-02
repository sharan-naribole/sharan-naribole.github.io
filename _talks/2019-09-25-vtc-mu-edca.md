---
title:  "802.11ax OFDMA talk at VTC 2019"
collection: talks
type: "Talk"
permalink: /talks/vtc-2019-mu-edca
date: 2019-09-25 12:00:00 -0600
redirect_from: 
  - /2019/09/25/vtc-2019-mu-edca.html
---
Presentation on 802.11ax OFDMA MU EDCA at [IEEE VTC Fall 2019](https://www.ieeevtc.org/vtc2019fall/)

## Abstract

To achieve increased radio utilization in dense WLAN deployments, IEEE 802.11ax amendment introduces Orthogonal 
Frequency Division Multiple Access to the Wi-Fi standard. As the Access Point (AP) initiates 
the uplink OFDMA operation via a trigger, this operation provides an additional means for the end-user devices (STAs) to gain
medium access. Consequently, to retain fairness in the network
and reduce contention, a novel MU EDCA channel access mode
has been introduced in 802.11ax standard to temporarily deprioritize
medium access for STAs participating in uplink OFDMA
operation. In this paper, we implement the key components of
802.11ax OFDMA operation in custom ns-3 network simulator
and evaluate the impact of MU EDCA channel access mode
on 802.11ax dense WLAN performance under various network
and traffic conditions. Our results show that (a) temporarily
switching to MU EDCA channel access mode significantly improves
network performance in terms of throughput and realtime
application latency (b) the duration of temporary switch to
MU EDCA channel access mode has negligible impact on network
performance and (c) 200 ms of MU EDCA channel access mode
can potentially support a dense network of almost 2000 STAs.

## Slides

[Download slides][slides]

<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/vtc_2019_mu_edca.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/vtc_2019_mu_edca.pdf