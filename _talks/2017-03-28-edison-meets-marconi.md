---
title:  "LiRa WLAN talk at SECON 2017"
collection: talks
type: "Talk"
permalink: /talks/secon-2017-lira
date: 2017-03-28 12:00:00 -0600
redirect_from: 
  - /017/03/28/edison-meets-marconi.html
---

In this blog, I give a high-level overview of LiRa paper accepted to IEEE International Conference on Sensing, Communication and Networking (SECON) 2017. The conference will be held June 12-14 in San Diego. SECON committee reviewed 170 submitted papers and accepted 45 for publication and presentation at the conference.

I attended IEEE SECON conference last year as well to present my paper on 60 GHz multicast titled **Scalable Multicast in Highly-Directional 60 GHz WLANs** ([IEEE listing][60ghz-multicast]). This conference was special because it was held in London :smile:

![IEEE SECON 2016,London](/images/secon_2016/secon_2016.jpg "IEEE SECON 2016, London")

<center> On top of London Eye. </center>

I shouldn't digress further. **Rice Engineering** publishing an article about my SECON 2017 paper motivated me to write this blog post. Link to the Rice Engineering article: [ECE students' paper accepted at IEEE conference][rice-engg].

## Paper

**LiRa: a WLAN architecture for Visible Light Communication with a Wi-Fi Uplink** was written by me and Shuqing “Erica” Chen and Yuqiang “Ethan” Heng, both seniors. All of us belong to the Rice Networks Group of Edward W. Knightly, professor, chair of ECE and co-author of the paper, who refers to the project as “Edison meets Marconi.”

## Visible Light Communication

Visible light communication (VLC) is an emerging technology that uses LED-based lighting for illumination and communication. Ceiling-mounted luminaries can modulate lighting in a manner undetectable by the human eye but that can be detected by mobile devices equipped with photo diodes surfaces. VLC-enabled luminaries can support low-rate “Internet of Things” applications and Gigabit-rate wireless networking for live HD video streaming.

[pureLiFi][pureLiFi] are currently the leaders in commercializing this technology. Prof. Harald Haas, Chair of Mobile Communications and co-founder of pure-LiFi coined the term **Li-Fi** (short form for Light-Fidelity). You must've realized the similarity with **Wi-Fi**, an abbreviation of "Wireless Fidelity".

**TED talks by Harald Haas**:

- [Wireless data from every light bulb][haas-2011]

- [Forget Wi-Fi. Meet the new Li-Fi Internet][haas-2015]

## Our Contributions

Unfortunately, the wide coverage and relatively high transmit power realized by the downlink to accommodate illumination is problematic to realize on the uplink. Even if a mobile client utilizes infrared LEDs, as pureLiFi does, providing wide aperture long-range transmission is ill-suited to mobile devices’ form factor and energy constraints.

*LiRa*, a Light-Radio wireless local area network proposed by us, combines light and radio links on a frame-by-frame basis at the MAC layer. In contrast to commercial systems, LiRa does not require uplink infrared transmission by the mobile client, and instead uses a radio uplink seamlessly integrated with legacy Wi-Fi.

## Uplink Challenge with Wi-Fi

In a traditional Wi-Fi network, packet error control is maintained by reserving the Wi-Fi channel access to user that just received a Wi-Fi data packet from the Access Point (AP). Using this reserved access, the reception of the data packet can be immediately acknowledged. In contrast, in the joint VLC and Wi-Fi network, when the user receives data over VLC, there might be ongoing Wi-Fi transmissions that prevent the user from immediately transmitting its feedback over Wi-Fi. After the Wi-Fi channel becomes free, the user competes against other users for channel access. This leads to significant increase in feedback delay and increased packet loss over Wi-Fi especially when there are many VLC users contending to transmit feedback.

## Protocol Design

To overcome this challenge, we designed AP-Spoofed Multi-client ARQ (ASMA) protocol. In ASMA, VLC user no longer have to contend on Wi-Fi to transmit feedback. Instead, the LiRa AP triggers the VLC users for the feedback in *configurable time intervals*. This trigger message is designed to include a spoofing mechanism that defers Wi-Fi users from accessing the channel while providing collision-free access to the VLC users to transmit their ACKs. In typical indoor environments, LiRa can achieve over 15x reduction in the feedback delay compared to the baseline solution of 802.11 contention. At the same time, LiRa has a negligible impact on Wi-Fi traffic performance.

## Significance

With our work, VLC can achieve its full potential through efficient error control feedback transmitted over Wi-Fi. Despite heavy internet traffic, our design enables smartphones to provide near-instant short Wi-Fi responses to maintain connection. For example, hospitals have constant heavy Internet traffic and large metallic interfering equipment that can prevent doctors from receiving real-time critical messages about patients’ health conditions. Doctors can rely on this technology to save time and save lives.

## Mentorship

Rice University is awesome! More than 60% of the undergrads have significant research experience by the time they graduate ([Source][rice-undergrads]). Erica and Ethan joined our group as summer interns in 2016. For the summer, they were heavily involved in the implementation of our VLC and radio testbed. For the VLC link, we studied the line-of-sight (LOS) and non-LOS VLC channel characterization in the presence of interfering light sources, mobility, rotation and blockage. For the radio link, Erica implemented the modified 802.11 contention procedures and our ASMA protocol. After this successful experience mentoring and working with undergrads, I plan to have another undergrad or two work with me on my final PhD thesis project.

## Conclusion

I am excited to visit San Diego :sunglasses: La Jolla Cove :heart:

Thanks for reading!

[pureLiFi]: http://purelifi.com/
[60ghz-multicast]:http://ieeexplore.ieee.org/document/7733014/
[rice-engg]: https://engineering.rice.edu/secon_conference_paper
[rice-undergrads]: https://engineering.rice.edu/
[haas-2011]: https://www.youtube.com/watch?v=NaoSp4NpkGg
[haas-2015]: https://www.youtube.com/watch?v=iHWIZsIBj3Q
