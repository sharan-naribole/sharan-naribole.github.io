---
title:  "LiRa WLAN talk at SECON 2017"
collection: talks
type: "Talk"
permalink: /talks/secon-2017-lira
date: 2017-06-14 12:00:00 -0600
redirect_from: 
  - /2017/06/14/secon-2017-lira.html
  - /2017/03/28/edison-meets-marconi.html
---

Presentation on LiRa WLAN at [IEEE SECON 2017](https://secon2017.ieee-secon.org/)

## Slides

[Download slides][slides]

<p align = "center">
<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/secon_2017_60GHz_lira.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>
</p>

### Slide 2: Visible Light Communication System

Visible Light Communication (VLC) is a key emerging communication technology that dual purposes LED-based lighting infrastructure for both illumination and communication. In particular, ceiling-mounted luminaries can modulate lighting in a manner unnoticeable to the human eye (i.e., flicker free) for reception at mobile devices fitted with low- cost photo diodes integrated with their casing surfaces. The proliferation of VLC-enabled luminaries promises not only support for low-rate IoT applications but also Gigabit rate wireless networking. Moreover, VLC has been demonstrated to enable higher resolution localization than radio.

### Slide 3: Infeasible VLC Uplink

Unfortunately, the wide coverage and relatively high transmit power realized by the downlink to satisfy the illumination objective is problematic to realize on the uplink. This is because even if a mobile client is fitted with infrared LEDs, providing wide aperture long-range transmission is ill-suited to mobile devices’ form factor and energy constraints. The narrow field-of-view also leads to the problem of rotational misalignment in the uplink. These uplink challenges faced by VLC systems can be addressed by utilizing radio band which not only provides wider coverage but also increased robustness to device rotation/mobility.

### Slide 4: Objective

The objective of this paper is to design, implement and evaluate a high performance WLAN system with VLC simplex downlink and RF in the uplink. At the same time, we want to retain the operation of legacy Wi-Fi for users who still access Wi-Fi as the only mode of downlink and uplink communication. As the Wi-Fi is being used for the VLC uplink, we want our system to have a controlled impact on the legacy WiFi performance. Let us look if prior works can satisfy these objectives?

### Slide 5: Prior Work

In prior works, VLC AP and Wi-Fi AP have been treated as separate devices connected at the networking layer and above. The main focus of these works has been the Downlink service i.e. load balancing between VLC and Wi-Fi or using the contention in Wi-Fi to determine the VLC downlink transmission order. However, none of these works have addressed the feedback channel for VLC via RF. So, can we simply extend the existing system for providing the feedback?

### Slide 6: Encapsulated handshake

The key methodology in Wi-Fi for error control is the Data- ACK Handshake procedure. In this simplistic illustration, once a data packet is transmitted, the channel is also reserved for a near-instant ACK feedback. Let’s see how this will work out in our VLC-WiFi WLAN. When a client receives VLC data packet in the downlink, it has to transmit the VLC MAC layer ACK in the WiFi channel. For the Wi-Fi AP to decode the VLC ACK packet and accordingly forward it to the VLC AP, VLC ACK needs to be encapsulated within a WiFi data frame. In this example, I have shown a simple scenario where where we have a single VLC client in the network and they are able to provide near instant feedback of VLC ACK on the WiFi channel.

### Slide 7: Encapsulated handshake

Due to LiRa’s co-existence with legacy Wi-Fi, there might be an ongoing transmission in the Wi-Fi channel that prevents a client from sending feedback immediately after VLC data reception. Consequently, LiRa cannot employ the 802.11-style two way DATA-ACK handshake for downlink VLC data as such an approach would incur severe ACK delays, orders of magnitude greater than SIFS, and would consume excessive resources on the RF network. Not only is there a delay but also because of the airtime spent in VLC uplink feedback and the collisions and contention, there is an uncontrollable degradation of the legacy Wi-Fi.

### Slide 8: LiRa: Light-Radio WLAN

In this paper, we design, analyze, and implement LiRa, a Light-Radio WLAN that fuses light and radio capabilities in an integrated system design without requiring mobile devices to emit light or infrared. We design an uplink control channel for LiRa that is Wi-Fi compliant, has a controllable impact on airtime taken from legacy Wi-Fi clients, and efficiently scales with increasing VLC user population. We implement LiRa’s key components and perform extensive over-the-air experiments. 

### Slide 9: LiRa Architecture

The key design principle is to enable AP-controlled feedback access instead of a per-client, independent contention. Yet, at the same time, we do not want to change how the radio MAC layer works because the legacy Wi-Fi users need to still communicate with the AP using 802.11 protocol. For this purpose, we integrate the VLC and Wi-Fi at the MAC layer. The key addition to the protocol stack is to have a single hardware interface in layer-2 for the higher layers. We have both VLC MAC and 802.11 MAC where VLC MAC is responsible for the scheduling and framing of the VLC frames. Below, these MAC layers is the LiRa aggregation and PHY-Adaptation layer. On the AP side, this is responsible for the VLC PHY-adaptation of the luminaries and MCS based on signal strength reports. 
At the client side, the role is quite different. Because client feedback is sent on-demand in response to the AP,  we opportunistically aggregate ACK information up to the time that the command is received by the client. When commanded, the client sends an aggregate ACK with a bitmap representation similar to Block ACK mechanism in Wi-Fi. The key difference with Block ACK is that the LiRa client does not negotiate a fixed block size with the AP to eliminate the negotiation overhead in multi-client feedback.

### Slide 10: AP-Controlled Feedback

The VLC downlink transmits multiple frames in succession that can span multiple LiRa clients before reception of acknowledgement feedback to control ARQ. As illustrated in the simplified timeline, VLC downlink frames are transmitted by one luminary or a group of luminaries. The AP is shown sequentially transmitting variable sized frames to different LiRa clients as indicated by the depicted numbers. 

### Slide 11: AP-Controlled Feedback

To ensure the channel access, the AP employs a prioritized access strategy as follows: When a channel becomes idle, based on the 802.11 standard, each user begins backoff only after the channel becomes idle for DIFS (= SIFS + 2*SLOT) duration, where SLOT refers to the length of one backoff counter slot duration. To guarantee that the AP acquires the channel before other users start backoff, the AP sends the ASMA trigger message after the channel becomes idle for PIFS (SIFS + SLOT) time. In case other APs are in range, a less aggressive policy can be used to minimize the chance of collisions of such messages. This is similar to how the Beacon frame is transmitted to begin the contention-free period in PCF and HCF.

### Slide 12: AP-Controlled Feedback

The objective of the ASMA trigger message is two fold: first, it protects the radio channel for a sufficient duration to enable transmission of the required feedback; second, it provides a transmission schedule for feedback such that multiple clients can send feedback messages without incurring per-client contention and collisions. 

### Slide 13: AP Trigger

We achieve the former objective via the virtual carrier sense mechanism of Wi-Fi in which other stations defer according to the duration contained in the header’s Network Allocation Vector (NAV). Namely, the LiRa AP spoofs the NAV: instead of inserting the AP’s actual frame transmission duration, it advertises a NAV to enable receipt of the feedback that the AP requires from the stations. 
Thus, the trigger message includes an identifier and start time for each LiRa client that the AP requires feedback from. The start time, expressed in mini-slots off-set from the end of the trigger message, enables a group of clients to transmit on the radio uplink sequentially without random access or polling. The figure’s timeline illustrates an example in which three clients are commanded by the trigger to transmit feedback, client 1 and 4, which have received data and will feedback both VLC ARQ feedback information and PHY updates, and client 3, which has not received data, but the AP may require other control information from, such as a PHY update to ensure that the client is matched with the best luminary. 

### Slide 14: Trigger Timer for controlled Wi-Fi impact 

To balance the LiRa responsiveness and control the airtime overhead on the Wi-Fi channel, ASMA sets a feedback trigger timer: once the timer expires, the AP will aggressively attempt to access the radio channel once it becomes idle in order to transmit an ASMA trigger message. In the paper, we show that the feedback trigger time shares an inverse relationship with the airtime overhead and a linear impact on response delay. Further, we evaluate policies for setting the trigger time in practice in our experimental evaluation. 

### Slide 15: Implementation

LiRa can employ any VLC physical layer. Here, we repurpose Philips Hue smart-LED lightbulbs that are capable of changing hues and light intensities as transmitters. While the Philips’ hardware and software interfaces do not allow high frequency modulation, the platform enables study of a large set of LiRa performance factors as described below. Most critically, the achievable data rate on a VLC link is a function of the received signal strength at the receiver’s photodiode. 
The LiRa VLC receiver employs Adafruit TSL2591 high dynamic range digital light sensors. We collect light intensity measurements at locations corresponding to different distances between the LED bulb and receivers spanning over 150 cm. At every location, the receiver’s sensor is rotated in steps of 10 degree.

To evaluate the impact of feedback trigger time, LiRa client size, and legacy user traffic characteristics, we collect over-the-air measurements using the software-defined radio WARP- based LiRa implementation. We perform independent reruns spanning several hours given a setting of LiRa trigger time, Wi-Fi operating channel and legacy user traffic flows. 

### Slide 17: Congested Channel Feedback Delay

In ASMA, the AP wins channel contention by transmitting the trigger message a duration PIFS after the channel becomes idle. However, the time between ASMA triggering the AP to enqueue the trigger message for transmission and the AP transmitting the trigger message over-the-air depends on the Wi-Fi traffic characteristics which includes uplink and downlink transmissions from legacy stations as well as uplink data from LiRa clients. Moreover, even when a controlled experiment is run with a fixed number of legacy users, there might be interference from legacy users associated with other APs nearby. For this purpose, we analyze the impact of Wi-Fi traffic on ASMA’s response delay in two dimensions: number of Wi-Fi traffic flows and the Wi-Fi operating channel. 
The metric response delay is the duration between when the AP transmits a VLC frame to a particular client and when the AP receives ARQ feedback for that particular frame. The mean response delay is an average over all frames and clients, as well as over time with multiple trigger messages. 

### Slide 18: Congested Channel Feedback Delay

First, independent of the number of traffic flows and the Wi-Fi channel, the mean response delay is consistently lower than the corresponding trigger time value. This is because frames transmitted in the latter part of the trigger duration typically have a response delay that is less than the trigger time itself (cf. Fig. 3). Second, there are differences in response delay across different channels due to uncontrolled Wi-Fi transmissions occurring in the range of the AP. The channels 14 and 48 have negligible variance in response delay because they are not used for commerical Wi-Fi operation in the experiment coverage area. Finally, response delay increases with the number of flows in Channel 1. This is due to the increased probability of the channel being occupied by legacy user transmissions when the trigger timer expires. In our experiments, we observe behavior similar to this figure for varying sets of LiRa client sizes and trigger times spanning from 1 ms (frequent feedback) to 10 ms (less frequent feedback). 

### Slide 19-20: Feedback with Baseline Strategy

To analyze the gains provided by ASMA’s strategies of spoofed NAV and multi-client scheduled feedback, we com- pare ASMA’s performance with an alternative strategy em- ploying client-driven feedback. We define Per-client Contention (PCC) as a feedback mechanism in which each VLC client independently contends via 802.11 to transmit feedback, such that VLC feedback is treated as an encapsulated Wi-Fi data frame. This approach contrasts with ASMA in that because the AP has not provided a spoofed NAV to allow feedback, the PCC clients providing feedback must contend independently. Consequently, the total feedback contention on the radio channel is once per client vs. once per trigger time. Nonetheless, we do not require PCC clients to contend for each downlink VLC frame. Instead, a PCC client begins contention as soon as it has feedback to send to minimize the response delay. 

### Conclusion

In this talk, I presented the design and implementation of LiRa, a WLAN that fuses simplex VLC downlink and bi-directional Wi-Fi on a frame-by-frame basis at the MAC layer. In order for LiRa clients to transmit VLC ARQ feedback via Wi-Fi without excessive contention-based delays, I presented ASMA as a scalable VLC feedback channel over RF. Using over-the-air measurements, in the paper, we demonstrate limitations of uplink coverage using infrared/VLC. Moreover, we show that compared to feedback using 802.11-based per-client contention, LiRa’s spoofed NAV and multi-client scheduled feedback reduces response delay by a factor of 15 and reduces degradation of Wi-Fi throughput to 3% from 74%. 

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/secon_2017_lira.pdf