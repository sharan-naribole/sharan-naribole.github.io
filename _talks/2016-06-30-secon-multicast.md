---
title:  "Scalable 60 GHz Multicast talk at SECON 2016"
collection: talks
type: "Talk"
permalink: /talks/secon-2016-60GHz-multicast
date: 2016-06-30 12:00:00 -0600
redirect_from: 
  - /2016/06/30/secon-2016-60GHz-multicast.html
---
Presentation on Scalable 60 GHz Multicast at [IEEE SECON 2016](https://secon2016.ieee-secon.org/)

## 1-min Video

<p align = "center">
<iframe width="560" height="315" src="https://www.youtube.com/embed/-bieBfboWNA" title="YouTube video player" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>
</p>


## Slides

[Download slides][slides]

<p align = "center">
<iframe src="https://docs.google.com/viewer?url=https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/secon_2016_60GHz_multicast.pdf&embedded=true" width="100%" height="600px" style="border:thick solid #708090 ;">Your browser does not support the PDF embedding. </iframe>
</p>


### Slide 2: 60 GHz Multicast 

In this talk, we focus on multicast service at 60 GHz. A multicast service provides multiple clients (a multicast group) with the same data from the Access Point (AP). The shown figure illustrates an indoor environment setting - it could be a hotspot where several users would like to access a particular TV channel such as Euro 2016 or Wimbledon stream and we are looking at new multimedia applications such as HD video streaming that typically require Gbps rates. Such high rates can be enabled by 60 GHz due to the huge bandwidth of 7-14 GHz available for unlicensed operations. However, the signal propagation at 60 GHz significantly differs from that of 2.4/5 GHz bands. The key difference being an increased signal attenuation of 20-40 dB depending on the environment. To overcome the signal attenuation, for a simple unicast transmission, one can using the narrowest beam possible to maximize the directivity gain. 802.11ad advocates beams as narrow as 3 degree using over 100-element antenna arrays. Can we simply extend this unicast scenario and transmit to all the multicast clients using parallel narrow beams?

### Slide 3: 60 GHz Multicast = Simple Extension to Unicast? 

Unfortunately, it turns out, the state-of-the art 60 GHz systems employ a single RF chain behind the entire antenna array due to power consumption constraints while operating over multi-gigahertz bandwidth. This means only a single beam can be generated at any time by the AP’s antenna array. This results in a switched beam system for multicast where the AP has to sequentially transmit the data via narrow beams to each client separately. Expectedly, the total time to transmit to all clients linearly increases with the multicast group size.

### Slide 4: Wide Beams

One way to reduce this time is to use the widest beams at the AP to cover many clients simultaneously. But, due to the increased attenuation at 60 GHz, not all the clients might be reachable using the widest beams. Even if they can be transmitted data, there is a fundamental tradeoff between the Beamwidth and the MCS, so the MCS selected for covering the client would have a low data rate leading to an inefficient multicast performance.

### Slide 5: Minimizing Total Transmission Time

Our objective for the data transmission is to minimize the total transmission time to send the data to all the clients. As illustrated in the figure, when two or more cleints are close to each other it might be possible to cover all of them using a single wide beam. However, we appreciate that the data rate used for this beam would be lower. In contrast, when clients are more isolated, it is imperative to use the narrowest beams to maximize the data rate. This kind of optimization of selecting the beams and their corresponding MCS is possible when the AP knows the per-beam per-client RSSI measurements. 

In particular, we propose the Scalable Directional Multicast protocol, the first 60 GHz multicast protocol to incorporate this overhead in the beam training and beam grouping.  Next, I give a brief overview of SDM’s key concepts.


### Slide 6: Overhead

Obtaining this information and performing the optimization involved an overhead that can significantly reduce the time for the actual data transmission. Let’s say the AP transmitted a training beacon using every beam that can be generated including wide beams and narrow beams and obtained feedback from every client, then the training overhead is exponential in terms of the no. of beamwidth levels for the AP’s codebook. Moreover, once the training information is obtained, performing a brute-force search to obtain the best set of beams and their MCS using the optimization discussed in the previous slide involves exponential overhead in terms of the number of codebook levels and a factorial order in terms of the multicast group size. In this paper, we proposed Scalable Directional Multicast or SDM for short, the first 60 GHz multicast protocol that incorporates the overhead in the training and beam grouping. Next, I am going to provide a brief overview of SDM’S key strategies.

### Slide 7: SDM Overview

The first key strategy is to utilize a multi-level codebook tree structure. Using basic physics of the transmit beam patterns, we can establish relationships between wide beams and fine beams and perform a pruned traversal of the codebook instead of doing an exhaustive training. Our key strategy to address unreachability of clients when wide beams are used is to begin with an exhaustive training from the finest level and progress to the wider beam levels using the only the parent beams in the codebook tree based on client’s feedback. I am also going to show you our solution for addressing cases of blockage and NLOS links. Now, with all this training info, the question of which wide beams can be possibly used for the data transmission still remains. We introduce a new metric Wide Beam Improvement that utilizes an only finest beams solution as baseline and quantifies the improvement in the transmission time separately made by every wide beam. This helps us in ranking the wide beams and accordingly add them to our final solution for the data transmission.

### Slide 8: Multi-Level Codebook Trees

Firstly, a multi-level codebook wasn’t required in a unicast transmission because we would like to use the narrowest beams for the transmission to maximize the directivity gain. For multicast, having multiple beamwidth levels provides flexibility for the AP to cover multiple clients simultaneously. When we perform training, we would like leverage the clients’ feedback information after each codebook level training to select only a partial set of beams to be used in the next level training. This is in comparison to performing an exhaustive training at each level. We use existing techniques of spatial similarity in transmit beam patterns to form edges between beams across different levels. This leads to a codebook tree where every beam has children in the immediate finer beam level and a parent in the immediate wider beam level. The figure shown below is an idealistic scenario where we can begin with exhaustive training at the widest beam level and based on client feedback, use only the children for the next beam level training instead of exhaustive training. However, this training approach faces challenges in practical scenarios.

### Slide 9: Challenges

First, every client might not be reachable at every level so when a client does not provide feedback after a codebook level training, the AP has no option but to choose all the beams of next level for training. In the worst case, this leads to an exhaustive training approach whose overhead is exponential in the number of codebook level. Second, as the codebook tree design implemented in the AP is independent of the environment in which the AP is deployed, there might be temporary reflectors/ blockage elements that make the codebook tree traversal imperfect. As shown in the bottom figure, a fine beam is reachable to the client through NLOS link but its parent wide beam in the tree is blocked by an object.

### Slide 10: SDM’s Finest Beam Training

SDM’s key strategy to address the unreachability challenge is to begin training at the finest level corresponding to the narrowest beams. By performing an exhaustive training with the beams of highest directivity gain, SDM ensures every client has at least one beam that could be used for data transmission. Using this feedback, the training progresses to wider beam levels using only the parent beams of the beams reported in feedback. Also, just using this training feedback, SDM obtains an initial beam group solution consisting of only the finest beams. This solutions serves as a baseline to how much reduction in transmission time can be obtained if a few finest level beams are replaced by a wide beam.

### Slide 11: SDM’s Wide Beam Training

For the wide beam training, SDM’s key strategy is to utilize only the parent beams in the codebook tree of the beams that were the best for each client in the fine beam level. Few clients might not provide any feedback as none of these beams were reachable either due to distance or due to blockage. To address blockage, SDM conducts an additional level of training using the sibling in the codebook tree of the beams that were initially selected for training. Through these strategies, SDM performs a pruned traversal of the codebook tree in contrast to exhaustive training. Our results in paper show the info collected is sufficient for near-optimal transmission performance.

### Slide 12: Which Wide Beams can be used?

To understand if a wide beam can be present in our final beam group solution or not, we begin with an initial solution composed of only finest level beams. Initial solution will not be the best solution if there exists at least one wide beam that can reduce the total transmission time. To rank all such possibilities, SDM’s key strategy is to introduce the metric called Wide Beam Improvement Ratio that finds out the extent of improvement a wide beam can provide in the total transmission time if this is the only wide beam present in the final solution replacing the finest beams that were initially serving the clients now served by the wide beam. Accordingly, SDM computes Wide Beam Improvement Ratio for each wide beam used in the training and adds them the solution until all clients are served by wide beams. It is possible clients that are isolated are still served by the finest beams in the final solution as shown in the bottom figure.

### Slide 13: Alternative Strategies

To compare SDM’s performance, we use two alternative strategies. Firstly, the only Finest Beams strategy conducts training only at the finest level and using that info generates the narrowest beams for each client. Second strategy includes an exhaustive training across all the codebook levels followed by an exhaustive search  for the best possible beam group solution. So, if we did not take into account the overhead in training and beam grouping, this would be the best solution.  

### Slide 14: Experimental Evaluation

To evaluate our protocol, we implement a 60 GHz system that consists of a RF frontend from Vubiq. For the baseband modulation and to streamline the measurement process, we integrate the frontend with WARP boards and the WARPLab framework. The measurements are collected in a typical conference room environment that includes a rich LOS and NLOS environment with several reflectors. The AP is placed at one end of the table and the client is placed in 10 different positions. To emulate blockage in practical scenarios from humans, laptops etc. we take measurements for three different orientation of client’s receiver. We feed the traces to a 60 GHz WLAN simulator that incorporates the 802.11ad packet size and timing parameters.

### Slide 15: Throughput Performance

We analyze the throughput performance incorporating overhead for training and beam grouping computation. The x-axis is the no. of clients in the multicast group and the y-axis is the throughput relative to the Exhaustive strategy. We select this as the baseline because this strategy selects the optimal set of beams for the data transmission. When there is a single client in the network, all strategies provide the same solution which is the narrowest beam. As the only finest strategy conducts the training only in one codebook level, it has more time for the data transmission hence provides the best performance. In contrast, the exhaustive has the worst performance as its training overhead is the highest.

### Slide 18: Related Work

Few works have presented algorithms for optimal beam scheduling [3]. With multiple RF chains, users can be localized in distance and angle and beams can be shaped non- symmetrically. In contrast, we focus on multicasting with a single-lobe pattern generation, as it requires only a single RF chain as all state-of-the-art commercial 802.11ad chipsets employ. 

Unicast Beamforming. A few recent works present solutions to reducing 60 GHz beam training overhead with the objective of establishing a fine beam unicast link. The protocol in [5] optimizes the codewords used in the wider beam levels using signal strength gradient change techniques. In our work, as the training is conducted for all clients at the same time, the gradient changes in the beacon signal strengths could be highly uncorrelated across the different clients thereby preventing gradient-change based optimization. The wider beam training is altogether skipped in [4] by training in legacy Wi-Fi band instead. In our work, the 60 GHz channel gain information even for a wide beam is important in finding an efficient beam group. 

### Slide 19: Conclusion

To conclude, our novel design SDM addresses the challenges imposed by directional communication for a scalable multicast service at 60 GHz. Using over-the-air measurements and trace driven simulations, I’ve shown that despite SDM not being exhaustive, it still collects sufficient information to generate a near-optimal beam group that leads to an overall better performance compared to exhaustive and narrow beam strategies. 

[slides]: https://github.com/sharan-naribole/sharan-naribole.github.io/raw/master/files/secon_2016_60GHz_multicast.pdf