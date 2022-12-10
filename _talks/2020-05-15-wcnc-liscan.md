---
title:  "LiSCAN talk at WCNC 2020"
collection: talks
type: "Talk"
permalink: /talks/wcnc-2020-liscan
date: 2020-05-15 12:00:00 -0600
redirect_from: 
  - /2020/05/15/wcnc-2020-liscan.html
---
Presentation on LiSCAN at [IEEE WCNC 2020](https://wcnc2020.ieee-wcnc.org/program/full-program) 

## Abstract

Contention-based uplink radio access might lead to significant degradation in airtime efficiency and 
energy efficiency as the time spent “awake” by the radio is dependent on the network traffic conditions. 
In this paper, we design and evaluate LiSCAN, a visible light uni-directional control channel for contention-free 
uplink radio access. LiSCAN enables a virtual full-duplex operation by broadcasting polling frames (light-polls) 
across distributed LED luminaires concurrently with data reception over radio. In LiSCAN, each client consists of 
an additional low-power light sensor, which upon hearing a light-poll directed to it by Access Point (AP), wakes 
up the radio module only if there is backlogged traffic. To maximize the airtime efficiency, LiSCAN transmits 
light-polls successively until it detects an uplink radio transmission. LiSCAN’s pipelined polling enables clients 
to detect failure in uplink packet reception and accordingly abort their transmissions to eliminate collisions at the AP.
We simulate LiSCAN and alternate strategies in ns-3 network simulator to analyze LiSCAN’s performance under varying 
traffic conditions. Our results show that LiSCAN can provide significant improvements in the radio airtime efficiency, 
the sessions delivered with pre-defined service quality requirements and energy savings.

## Slides

[Download slides][slides]

<p align = "center">
<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/wcnc_2020_liscan.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>
</p>


### Slide 1: Dense Wireless Sensor Networks

Sensors have becoming a leading solution in many important applications such as industrial automation, smart buildings, telemedicine, etc. A wireless sensor network typically consists of a large number of small, low-cost sensor nodes distributed in the target area for collecting data of interest. The data flow in such networks is mainly in the uplink direction from the sensors to the Access Point. These sensors are often powered with batteries, therefore energy-efficiency is extremely important. Ideally, we would like a sensor to be awake only for its data transmission and return to sleep state immediately. However, as shown in the illustration, using contention-based access, after a sensor wakes up upon collecting new data, it might have to wait for a long time before it can transmit. 
In an ideal scenario, the AP has perfect knowledge of when the sensors would generate traffic. In such a scenario, the AP can trigger a sensor to turn on its radio only when the sensor has backlogged traffic leading to the sensor transmitting its backlogged data immediately without contention. Unfortunately, the traffic generation at the sensors follow asynchronous patterns. Therefore, the AP lacks perfect knowledge of each sensor’s backlogged traffic. Instead of making the operation efficient,such control triggers to hundreds of sensors might consume significant airtime over the radio band as majority of the sensors being polled might not have any backlogged data.

### Slide 2: VLC Contention Free Access

Therefore, our key strategy is to transfer the AP’s polling to an out-of-band channel. In such a system, the sensor’s radio module can be turned on only for the data transmission. Due to the severe spectrum scarcity in Wi-Fi bands, sensors’ antenna, and power limitations, utilizing an additional radio channel exclusively for polling control becomes infeasible. In contrast, we utilize visible light communication or VLC for short for uni-directional out-of-band control. VLC is well-suited for energy-efficient operation by dual purposing LED-based lighting infrastructure for both illumination and communication. In particular, ceiling-mounted luminaries can modulate lighting in a manner unnoticeable to the human eye. These signals can be received at sensors fitted with energy-autonomous low-power wake-up VLC receivers that consume negligible energy from the sensor’s battery. 


### Slide 3: VLC Contention-Free Access - Energy Consumption

Using the VLC control channel, a sensor does not need it turn on its radio module when it generates data. Instead, it can wait for a light poll from the AP and turn on its radio module only for the data transmission. In this manner, energy consumption at sensor can be minimized.


### Slide 4: LiSCAN

In this paper, our objective is to maximize the uplink throughput incorporating the channel access delay and the energy consumption by the sensors. To achieve the objective, we propose LiSCAN, a visible light communication (VLC) uni-directional control channel for contention-free uplink radio access. Architecturally, LiSCAN is a new MAC sub-layer that a) integrates the VLC and Wi-Fi MAC layers at AP and Sensors and b) serves as a single layer-2 interface to the VLC and Wi-Fi PHY modules. We discuss the motivation and design of LiSCAN protocol in the next few slides.

### Slide 5 and 6: Polling in Contention-Free Access

Due to the asynchronous traffic generation at sensors, AP does not know which sensors currently have traffic. In the RF only case, after an AP sends a poll to sensor A, it has to wait for certain time to detect possible uplink transmission. Upon not hearing anything, it can with poll the next sensor. In contrast, using a VLC control channel, we can reduce the overhead by transmitting the next Light poll immediately after. This works out well in this example where there was no transmission from sensor A.

However, if both sensor A and Sensor B had backlogged data, such pipelined polls can lead to collision at the AP as shown in the right figure.

### Slide 7: LiSCAN Pre-emptive Collision Avoidance

As soon as the AP detects the PHY preamble of the current sensor’s transmission, the AP will immediately abort the light-poll transmission. As the light poll transmission time is longer than the 802.11 packet detection, the light poll to next sensor is not fully transmitted before abortion. Consequently, the next sensor does not transmit any backlogged data and collision at AP is avoided.

### Slide 8: LiSCAN ACK over VLC

In LiSCAN, we utilize VLC instead of radio for ACK transmissions corresponding to successful uplink data reception. This serves two purposes: First, by transmitting over VLC, LiSCAN eliminates the radio airtime lost in AP switching from receive mode to transmit mode for ACK transmission and back to receive mode for future data reception. Second, by transmitting over VLC, the next sensor in schedule can transmit backlogged data over radio concurrently with ACK transmission over VLC. To achieve this, LiSCAN utilizes the PHY header of ongoing uplink data transmission to find out the time at which this transmission ends. As the light-poll to next sensor in schedule was aborted upon hearing the ongoing transmission’s preamble, LiSCAN begins the retransmission of next sensor’s light-poll to end at the same time as ongoing uplink transmission. In this manner, if the next sensor has backlogged data, it can begin the data transmission as soon as it decodes the light-poll. Concurrently, the AP can transmit the ACK if it successfully receives the uplink data frame. Accordingly, the next light-poll will be transmitted after the ACK transmission. 

### Slide 9: LiSCAN Evaluation

To analyze performance for varying network conditions, we extend the ns-3 network simulator with key components of LiSCAN and alternative strategies. To simulate realistic Internet traffic, we utilize the Poisson Pareto Burst Process (PPBP) model that has been validated to match the statistical properties of real-life IP networks. We consider a network of 100 sensors with a varying number of sensors actively generating traffic and also varying burst arrival rates. In each contention-free period for both LiSCAN and Contention-free radio access, the polling is performed in a round-robin manner and the schedule is selected randomly using a uniform distribution. For contention-based access, we utilized the existing ns-3 WiFi Radio Energy module to capture energy consumption. For the contention-free strategies, the power consumption in different states of sensor was modeled utilizing existing research works referenced in the paper.

### Slide 10: Energy Consumption

First, we analyze the energy consumption. Each sub-plot corresponds to a particular ratio of the sensors generating any traffic. The x-axis in each sub-plot represents the varying mean burst arrival rate at the sensors generating traffic during a runtime of 100 milliseconds. First, compared to LiSCAN, the radio-only strategies have significantly higher awake time. This is because the radio
of a sensor is on (a) when it has backlogged traffic, (b) uplink transmission and (c) ACK reception. Second, for low and moderate sensor ratios, the contention-based approach has higher awake time during high traffic rates because of increased waiting time during contention. Third, when the sensor ratio is high and the traffic rate is high, we observe both the contention-based and contention-free strategies converge in their radio awake time. This is because of the increased data generation in the radio only contention-free strategy. In contrast, in LiSCAN, the radio module of a sensor is turned on only for the uplink data transmission. Even if the sensor has backlogged traffic, the wake up receiver keeps the radio off until receiving a light-poll intended for this sensor. Moreover, the VLC wake-up receiver receives the ACK for a just concluded uplink data transmission instead of the sensor’s radio. LiSCAN’s utilization of VLC channel for radio control results in near-zero energy consumption on the radio channel.

### Slide 11: Aggregate Throughput

Next, we analyze the impact of the sensor ratio and the traffic arrival rate on the aggregate uplink throughput at the AP. With low sensor ratio, independent of the traffic burst rate, the contention-based access provides the highest throughput. This is because the contention-based strategies spend significant airtime polling sensors with no backlogged traffic. With moderate to high traffic rates, LiSCAN provides significant improvement in throughput compared to other strategies. Unlike contention-based access, the sensors in LiSCAN do not contend and transmit their data as soon as they receive a light-poll intended for them. Unlike radio-only contention, the increased radio utilization enables LiSCAN to transmit the light-polls and ACKs over VLC concurrently with uplink radio transmissions. In this manner, LiSCAN increases the radio airtime for uplink data transmissions compared to radio-only contention-free strategy. 

### Slide 12: Related Work

In our previous work, we designed LiRa, the first system to integrate VLC and RF at the MAC layer, and the first system to provide a virtual feedback channel for VLC via Wi-Fi. In contrast, LiSCAN is the first system to design a scalable uni-directional VLC control channel for contention-free RF uplink access. In-band asynchronous energy-saving MAC protocols have been proposed in literature. To further reduce the energy consumption, several works have proposed the use of additional low-power radio to aid the sensor’s wake up First, due to the spectrum scarcity, the efficiency of this operation might be significantly degraded in dense deployments. Second, a wakeup radio typically remains always active and consume energy from the attached sensor. In contrast, the illumination coverage of LED luminaires naturally provides improved spatial reuse even in dense deployments and the energy-autonomous nature of the wake-up VLC receivers leads to negligible impact on the sensor’s energy.

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/wcnc_2020_liscan.pdf