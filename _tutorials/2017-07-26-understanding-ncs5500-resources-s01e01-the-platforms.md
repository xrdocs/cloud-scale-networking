---
published: false
date: '2017-07-26 16:44 +0200'
title: Understanding NCS5500 Resources (S01E01 - The platforms)
author: Nicolas Fevrier
excerpt: First episode of a series of posts on NCS 5500 memory resources
tags:
  - ncs5500
  - LEM
  - LPM
  - TCAM
  - eTCAM
  - ncs 5500
position: hidden
---
This series of posts aim at explaining in details how NCS5500 routers use the different memory resources available for each type of features or prefixes. We will start describing the hardware implementation then we will explain how “databases” are used, how are working the profiles, what should be monitored and how you can troubleshoot it.
  
## NCS5500 Portfolio
  
Routers in the NCS5500 portfolio offer diverse form-factors. Some are fixed (1RU, 2RU), some others are modular (4-slot, 8-slot, 16-slot) with different types of line cards.
  
In August 2017, with one exception covered in a following xrdocs post, all of them are using Qumran-MX or Jericho forwarding ASICs (FA). Qumran is used for System-on-Chip (SoC) routers like NCS5501 and NCS5501-SE, all others are used in conjunction with one or several Fabric Engine.
  
We can categorize these systems and line cards in two families:

![Base vs Scale]({{site.baseurl}}/images/base-scale.jpg)

  
### Using external TCAM (named “Scale" and identified with -SE in the product ID)
  
- NCS5501-SE
  
![NCS5501-SE]({{site.baseurl}}/images/5501-se.jpg)
  
- NCS5502-SE
  
![NCS5502-SE]({{site.baseurl}}/images/5502.jpg)
  
- NC55-24X100G-SE
  
![NC55-24X100G-SE]({{site.baseurl}}/images/24x100-se.jpg)
  
- NC55-24H12F-SE
  
![NC55-24H12F-SE]({{site.baseurl}}/images/24h12f-se.jpg)
  
### Not using external TCAM but only the memories inside the FA (named “Base")
  
- NCS5501
  
![NCS5501]({{site.baseurl}}/images/5501.jpg)
  
- NCS5502
  
![NCS5502]({{site.baseurl}}/images/5502.jpg)
  
- NC55-36X100G
  
![NC55-36X100G]({{site.baseurl}}/images/36x100.jpg)
  
- NC55-18H18F
  
![NC55-18H18F]({{site.baseurl}}/images/18h18f.jpg)
  
- NC55-36x100G-S (MACsec card)
  
![NC55-36X100G-S]({{site.baseurl}}/images/36x100MACsec.jpg)
  
- NC55-6X200-DWDM-S (Coherent card)
  
![NC55-6X200G-DWDM-S]({{site.baseurl}}/images/6x200 COH.jpg)

  
**Note**: Inside a modular chassis, we can mix and match eTCAM and non-eTCAM line cards. A feature is available to decide where the prefixes should be programmed (differentiating IGP and BGP, and using specific ext-communities).
  
This external memory is used to extend the scale in term of routes and classifiers (Access-lit entries for instance).
It’s not a field-replaceable part, that means you can not convert a NC55-36X100G non-eTCAM card into an eTCAM card.
We have two eTCAM per FA and they are soldered to the board.
  
eTCAM should not be confused with the 4GB external packet buffer which is present on the side of each FA, regardless the type of system or line card. The eTCAM only handles prefixes and ACEs, not packets. The external packet buffer will be used in case of queue congestion only. It’s a very rapid graphical memory specifically used for packets.
  
## Resources / Memories
  
Each forwarding ASIC is made of two cores (0 and 1). Also we have an ingress and an egress pipeline. 
Each pipeline itself is made of different blocks. 
For clarity and IP reason, we will simplify the description and represent the series of blocks as just a Packet Processor (PP) and a Traffic Manager (TM).
  
![Resources]({{site.baseurl}}/images/resources.jpg)

  
Along the pipeline, the different blocks can access (read or write) different “databases”.
These databases are memory entities used to store specific type of information.
In follow up posts, we will describe in details how are they used, but let’s introduce them right now.
  
- The Longest Prefix Match Database (LPM sometimes referred as KAPS for KBP Assisted Prefix Search, KBP being itself Knowledge Based Processor) is an SRAM used to store IPv4 and IPv6 prefixes. It’s an algorithmic memory qualified for 256k entries IPv4 and 128k entries IPv6 in the worst case. We will see it can go much higher with internet distribution.
- The Large Exact Match Database (LEM) is used to store IPv4 and IPv6 routes also, plus MAC addresses and MPLS labels. It scales to 786k entries
- The Internal TCAM (iTCAM) is used for Packet classification (ACL, QoS) and is 48k entries large.
- The FEC database is used to store NextHop (128k entries), containing also the FEC ECMP (4k entries)
- Egress Encapsulation DB (EEDB) is used for egress rewrites (96k entries), including adjacency encapsulation like link-local details from ARP, ND and for MPLS labels or GRE headers
  
All these databases are present inside the Forwarding ASIC.
  
- The external TCAMs (eTCAM) are only present in the -SE line cards and systems and, as the name implies, are not a resource present on the Forwarding ASIC but on the side of it. They are used to extend unicast route and ACL / classifiers scale (up to 2M IPv4 entries)
  
In the next post, we will explain the way we are sorting and storing routes in LEM, LPM or eTCAM. Stay tuned for the next episode.