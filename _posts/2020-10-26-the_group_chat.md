---
layout: post
title: The Group Chat
description: Test
---

<p class="message">
  This challange asked me to analyze a wireshark capture of communications between a terroist group and an undercover agent.
  I forgot to grab screen captures of the answers being validated for all questions past the first but I'll do my best to describe them.
</p>

**Q1**\
Looking at the first packet shows that they are communicating on an Internet Chat Relay.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/network_traffic_analysis/thegroupchat_q1_screenshot.png">
<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/network_traffic_analysis/thegroupchat_q1_proof.png">

**Q2**\
The response packet from the server reveals the username of the undercover agent in the welcome message.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/network_traffic_analysis/thegroupchat_q2_screenshot.png">

**Q3**\
The username of the terrorist can be found in any of the response packets.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/network_traffic_analysis/thegroupchat_q3_screenshot.png">

**Q4**
The two are using the general channel to communicate.

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/network_traffic_analysis/thegroupchat_q3_screenshot.png">

**Q5**
The hand off will occur at "The Red Dragon"

<img src="https://raw.githubusercontent.com/lukej2680/lukej2680.github.io/master/_images/ncl_fall2020/network_traffic_analysis/thegroupchat_q4_screenshot.png">
