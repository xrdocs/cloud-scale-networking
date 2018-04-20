---
published: true
date: '2018-02-15 11:17 +0100'
title: Port Assignments on NCS5500 Platforms
author: Nicolas Fevrier
excerpt: Brief post on port allocation / NPU for NCS5500
tags:
  - NCS5500
  - ncs 5500
  - Port
position: top
---
{% include toc icon="table" title="Port Assignments on NCS5500" %} 

## Introduction

This short post will help understanding how the ports are allocated to NPU for each line card and systems.
It will be useful for future post(s) and particularly on topics like Netflow/IPFIX.

For example, it's important to understand the way our ports are distributed among forwarding ASICs when considering the amount of sample traffic from an NPU to the LC CPU is shaped 133Mbps or 200Mbps.

![sample.jpg]({{site.baseurl}}/images/sample.jpg)

Let's review platform by platform and line card by line card, how we do this allocation.
The following CLI is used to identify the port assignment, looking at the "local" VOQ port type.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:Router#show contr npu voq-usage interface all instance 1 location 0/7/CPU0

-------------------------------------------------------------------
Node ID: 0/7/CPU0
Intf         Intf     NPU NPU  PP   Sys   VOQ   Flow   VOQ    Port
name         handle    #  core Port Port  base  base   port   speed
             (hex)                                     type   (Gbps)
----------------------------------------------------------------------
Hu0/7/0/9    3800278   1   0    9  1833  12104  25928 local   100
Hu0/7/0/8    3800280   1   1   17  1841  12168  25992 local   100
Hu0/7/0/7    3800288   1   1   13  1837  12136  25960 local   100
Hu0/7/0/6    3800290   1   1   21  1845  12200  26024 local   100
Hu0/7/0/10   38001a8   1   0    5  1829  12072  25896 local   100
Hu0/7/0/11   38001b0   1   0    1  1825  12040  25864 local   100
...
RP/0/RP0/CPU0:Router#
</code>
</pre>
</div>

## Port allocation

### NCS5501(-SE) (Base and Scale version)

NCS5501 and NCS5501-SE are using a single Qumran-MX ASIC and all the SFP ports are connected to core 0 while all QSFP ports are connected to core 1.

![NCS5501-base.jpg]({{site.baseurl}}/images/NCS5501-base.jpg) ![NCS5501-scale.jpg]({{site.baseurl}}/images/NCS5501-scale.jpg)

### NCS5502(-SE) (Base and Scale version)

NCS5502s are made of 8 Jericho ASICs interconnected with 2x fabric engine (FE3600)

![NCS5502-scale.jpg]({{site.baseurl}}/images/NCS5502-scale.jpg) ![NCS5502-base.jpg]({{site.baseurl}}/images/NCS5502-base.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
|  Hu0/0/0/0 | 0 / 1 | Hu0/0/0/12 | 2 / 1 | Hu0/0/0/24 | 4 / 1 | Hu0/0/0/36 | 6 / 1 |
| Hu0/0/0/1 | 0 / 1 | Hu0/0/0/13 | 2 / 1 | Hu0/0/0/25 | 4 / 1 | Hu0/0/0/37 | 6 / 1 |
| Hu0/0/0/2 | 0 / 1 | Hu0/0/0/14 | 2 / 0 | Hu0/0/0/26 | 4 / 1 | Hu0/0/0/38 | 6 / 1 |
| Hu0/0/0/3 | 0 / 0 | Hu0/0/0/15 | 2 / 0 | Hu0/0/0/27 | 4 / 0 | Hu0/0/0/39 | 6 / 0 |
| Hu0/0/0/4 | 0 / 0 | Hu0/0/0/16 | 2 / 0 | Hu0/0/0/28 | 4 / 0 | Hu0/0/0/40 | 6 / 0 |
| Hu0/0/0/5 | 0 / 0 | Hu0/0/0/17 | 2 / 0 | Hu0/0/0/29 | 4 / 0 | Hu0/0/0/41 | 6 / 0 |
| Hu0/0/0/6 | 1 / 1 | Hu0/0/0/18 | 3 / 1 | Hu0/0/0/30 | 5 / 1| Hu0/0/0/42 | 7 / 1 |
| Hu0/0/0/7 | 1 / 1 | Hu0/0/0/19 | 3 / 1 | Hu0/0/0/31 | 5 / 1 | Hu0/0/0/43 | 7 / 1 |
| Hu0/0/0/8 | 1 / 1 | Hu0/0/0/20 | 3 / 1 | Hu0/0/0/32 | 5 / 1 | Hu0/0/0/44 | 7 / 1 |
| Hu0/0/0/9 | 1 / 0 | Hu0/0/0/21 | 3 / 0 | Hu0/0/0/33 | 5 / 0 | Hu0/0/0/45 | 7 / 0 |
| Hu0/0/0/10 | 1 / 0 | Hu0/0/0/22 | 3 / 0 | Hu0/0/0/34 | 5 / 0 | Hu0/0/0/46 | 7 / 0 |
| Hu0/0/0/11 | 1 / 0 | Hu0/0/0/23 | 3 / 0 | Hu0/0/0/35 | 5 / 0 | Hu0/0/0/47 | 7 / 0 |

### NCS55A1-24H

NCS55A1-24H is made of two Jericho+ connected back-to-back (no fabric engine)

![NCS5500-24H.jpg]({{site.baseurl}}/images/NCS5500-24H.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Hu0/0/0/0 | 0 / 1 | Hu0/0/0/9 | 0 / 0 | Hu0/0/0/18 | 1 / 1 |
| Hu0/0/0/1 |0 / 0 | Hu0/0/0/10 | 0 / 1 | Hu0/0/0/19 | 1 / 0 |
| Hu0/0/0/2 | 0 / 1 | Hu0/0/0/11 | 0 / 0 | Hu0/0/0/20 | 1 / 1 |
| Hu0/0/0/3 | 0 / 0 | Hu0/0/0/12 | 1 / 1 | Hu0/0/0/21 | 1 / 0 |
| Hu0/0/0/4 | 0 / 1 | Hu0/0/0/13 | 1 / 0 | Hu0/0/0/22 | 1 / 1 |
| Hu0/0/0/5 | 0 / 0 | Hu0/0/0/14 | 1 / 1 | Hu0/0/0/23 | 1 / 0 |
| Hu0/0/0/6 | 0 / 1 | Hu0/0/0/15 | 1 / 0 |   |   | 
| Hu0/0/0/7 | 0 / 0 | Hu0/0/0/16 | 1 / 1 |   |   |  
| Hu0/0/0/8 | 0 / 1 | Hu0/0/0/17 | 1 / 0 |   |   |

### NCS55A1-36H(-SE)

NCSA1-36Hs are made of 4 Jericho+ ASICs interconnected through a FE3600 ASIC.

![NCS5500-36H.jpg]({{site.baseurl}}/images/NCS5500-36H.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Hu0/0/0/1 | 0 / 0 | Hu0/0/0/9 | 1 / 0 | Hu0/0/0/18 | 2 / 0 | Hu0/0/0/27 | 3 / 0 |
| Hu0/0/0/1 | 0 / 0 | Hu0/0/0/10 | 1 / 0 | Hu0/0/0/19 | 2 / 0 | Hu0/0/0/28 | 3 / 0 |
| Hu0/0/0/2 | 0 / 0 | Hu0/0/0/11 | 1 / 0 | Hu0/0/0/20 | 2 / 0 | Hu0/0/0/29 | 3 / 0 |
| Hu0/0/0/3 | 0 / 0 | Hu0/0/0/12 | 1 / 0 | Hu0/0/0/21 | 2 / 0 | Hu0/0/0/30 | 3 / 0 |
| Hu0/0/0/4 | 0 / 1 | Hu0/0/0/13 | 1 / 1 | Hu0/0/0/22 | 2 / 1 | Hu0/0/0/31 | 3 / 1 |
| Hu0/0/0/5 | 0 / 1 | Hu0/0/0/14 | 1 / 1 | Hu0/0/0/23 | 2 / 1 | Hu0/0/0/32 | 3 / 1 |
| Hu0/0/0/6 | 0 / 1 | Hu0/0/0/15 | 1 / 1 | Hu0/0/0/24 | 2 / 1 | Hu0/0/0/33 | 3 / 1 |
| Hu0/0/0/7 | 0 / 1 | Hu0/0/0/16 | 1 / 1 | Hu0/0/0/25 | 2 / 1 | Hu0/0/0/34 | 3 / 1 |
| Hu0/0/0/8 | 0 / 1 | Hu0/0/0/17 | 1 / 1 | Hu0/0/0/26 | 2 / 1 | Hu0/0/0/35 | 3 / 1 |

### NCS55-36X100G and NC55-36X100G-S

In these cards we have 6 Jericho ASICs.

![NC55-36X100G.jpg]({{site.baseurl}}/images/NC55-36X100G.jpg)
![NC55-36X100G-S.jpg]({{site.baseurl}}/images/NC55-36X100G-S.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Hu0/x/0/0 | 0 / 1 | Hu0/x/0/9 | 1 / 0 | Hu0/x/0/18 | 3 / 1 | Hu0/x/0/27 | 4 / 0
| Hu0/x/0/1 | 0 / 1 | Hu0/x/0/10 | 1 / 0 | Hu0/x/0/19 | 3 / 1 | Hu0/x/0/28 | 4 / 0 
| Hu0/x/0/2 | 0 / 1 | Hu0/x/0/11 | 1 / 0 | Hu0/x/0/20 | 3 / 1 | Hu0/x/0/29 | 4 / 0
| Hu0/x/0/3 | 0 / 0 | Hu0/x/0/12 | 2 / 1 | Hu0/x/0/21 | 3 / 0 | Hu0/x/0/30 | 5 / 1
| Hu0/x/0/4 | 0 / 0 | Hu0/x/0/13 | 2 / 1 | Hu0/x/0/22 | 3 / 0 | Hu0/x/0/31 | 5 / 1
| Hu0/x/0/5 | 0 / 0 | Hu0/x/0/14 | 2 / 1 | Hu0/x/0/23 | 3 / 0 | Hu0/x/0/32 | 5 / 1
| Hu0/x/0/6 | 1 / 1 | Hu0/x/0/15 | 2 / 0 | Hu0/x/0/24 | 4 / 1 | Hu0/x/0/33 | 5 / 0
| Hu0/x/0/7 | 1 / 1 | Hu0/x/0/16 | 2 / 0 | Hu0/x/0/25 | 4 / 1 | Hu0/x/0/34 | 5 / 0
| Hu0/x/0/8 | 1 / 1 | Hu0/x/0/17 | 2 / 0 | Hu0/x/0/26 | 4 / 1 | Hu0/x/0/35 | 5 / 0

### NCS55-24X100G-SE

The scale 24x100G are made of 4 Jericho ASICs.

![NC55-24X100G-SE.jpg]({{site.baseurl}}/images/NC55-24X100G-SE.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Hu0/x/0/0 | 0 / 1 | Hu0/x/0/6 | 1 / 1 | Hu0/x/0/12 | 2 / 1 | Hu0/x/0/18 | 3 / 1 |
| Hu0/x/0/1 | 0 / 1 | Hu0/x/0/7 | 1 / 1 | Hu0/x/0/13 | 2 / 1 | Hu0/x/0/19 | 3 / 1 |
| Hu0/x/0/2 | 0 / 1 | Hu0/x/0/8 | 1 / 1 | Hu0/x/0/14 | 2 / 1 | Hu0/x/0/20 | 3 / 1 |
| Hu0/x/0/3 | 0 / 0 | Hu0/x/0/9 | 1 / 0 | Hu0/x/0/15 | 2 / 0 | Hu0/x/0/21 | 3 / 0 |
| Hu0/x/0/4 | 0 / 0 | Hu0/x/0/10 | 1 / 0 | Hu0/x/0/16 | 2 / 0 | Hu0/x/0/22 | 3 / 0 |
| Hu0/x/0/5 | 0 / 0 | Hu0/x/0/11 | 1 / 0 | Hu0/x/0/17 | 2 / 0 | Hu0/x/0/23 | 3 / 0 |

### NCS55-18H18F

By default, the base combo card offers 36 ports 40G, and it's possible to upgrade half of them to 100G.
This line card is made of 3 Jericho ASICs.

![NC55-18H18F.jpg]({{site.baseurl}}/images/NC55-18H18F.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Fo0/x/0/0 | 0 / 0 | Hu0/x/0/9 | 0 / 0 | Hu0/x/0/18 | 1 / 1 | Fo0/x/0/27 | 2 / 1 |
| Fo0/x/0/1 | 0 / 0 | Hu0/x/0/10 | 0 / 0 | Hu0/x/0/19 | 1 / 1 | Fo0/x/0/28 | 2 / 0 |
| Fo0/x/0/2 | 0 / 1 | Hu0/x/0/11 | 0 / 0 | Hu0/x/0/20 | 1 / 1 | Fo0/x/0/29 | 2 / 1 |
| Fo0/x/0/3 | 0 / 1 | Fo0/x/0/12 | 1 / 0 | Hu0/x/0/21 | 1 / 0 | Hu0/x/0/30 | 2 / 1 |
| Fo0/x/0/4 | 0 / 0 | Fo0/x/0/13 | 1 / 0 | Hu0/x/0/22 | 1 / 0 | Hu0/x/0/31 | 2 / 1 |
| Fo0/x/0/5 | 0 / 1 | Fo0/x/0/14 | 1 / 1 | Hu0/x/0/23 | 1 / 0 | Hu0/x/0/32 | 2 / 1 |
| Hu0/x/0/6 | 0 / 1 | Fo0/x/0/15 | 1 / 1 | Fo0/x/0/24 | 2 / 0 | Hu0/x/0/33 | 2 / 0 |
| Hu0/x/0/7 | 0 / 1 | Fo0/x/0/16 | 1 / 0 | Fo0/x/0/25 | 2 / 0 | Hu0/x/0/34 | 2 / 0 |
| Hu0/x/0/8 | 0 / 1 | Fo0/x/0/17 | 1 / 1 | Fo0/x/0/26 | 2 / 1 | Hu0/x/0/35 | 2 / 0 |

### NCS55-24H12F-SE

By default, the scale combo card offers 36 ports 40G, and it's possible to upgrade two third of them to 100G.
This line card is made of 4 Jericho ASICs with eTCAM.

![NC55-24H12F-SE.jpg]({{site.baseurl}}/images/NC55-24H12F-SE.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Fo0/x/0/0 | 0 / 0  | Fo0/x/0/9 | 1 / 1 | Fo0/x/0/18 | 2 / 0 | Fo0/x/0/27 | 3 / 0 |
| Fo0/x/0/1 | 0 / 0 | Fo0/x/0/10 | 1 / 0 | Fo0/x/0/19 | 2 / 0 | Fo0/x/0/28 | 3 / 0 |
| Hu0/x/0/2 | 0 / 1 | Fo0/x/0/11 | 1 / 0 | Hu0/x/0/20 | 2 / 1| Fo0/x/0/29 | 3 / 1 |
| Hu0/x/0/3 | 0 / 1 | Hu0/x/0/12 | 1 / 1 | Hu0/x/0/21 | 2 / 1 | Hu0/x/0/30 | 3 / 1 |
| Hu0/x/0/4 | 0 / 1| Hu0/x/0/13 | 1 / 1 | Hu0/x/0/22 | 2 / 1 | Hu0/x/0/31 | 3 / 1 |
| Hu0/x/0/5 | 0 / 0 | Hu0/x/0/14 | 1 / 1 | Hu0/x/0/23 | 2 / 0 | Hu0/x/0/32 | 3 / 1 |
| Hu0/x/0/6 | 0 / 0 | Hu0/x/0/15 | 1 / 0 | Hu0/x/0/24 | 2 / 0 | Hu0/x/0/33 | 3 / 0 |
| Hu0/x/0/7 | 0 / 0 | Hu0/x/0/16 | 1 / 0 | Hu0/x/0/25 | 2 / 0 | Hu0/x/0/34 | 3 / 0 |
| Fo0/x/0/8 | 0 / 1 | Hu0/x/0/17 | 1 / 0 | Fo0/x/0/26 | 2 / 1 | Hu0/x/0/35 | 3 / 0 |


### NCS55A2-MOD(-SE)-S


2RU chassis made of a single Jericho+ ASIC.

![55A2-MOD.jpg]({{site.baseurl}}/images/55A2-MOD.jpg)


| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Te0/x/0/0 | 0 / 0 | Te0/x/0/14 | 0 / 1  | TF0/x/0/28 | 0 / 0 | Hu0/x/1/2/0 | 0 / 1 |
| Te0/x/0/1 | 0 / 0 | Te0/x/0/15 | 0 / 1 | TF0/x/0/29 | 0 / 0 | Te0/x/2/0 | 0 / 1 |
| Te0/x/0/2 | 0 / 0 | Te0/x/0/16 | 0 / 0 | TF0/x/0/30 | 0 / 0 | Te0/x/2/1 | 0 / 0 |
| Te0/x/0/3 | 0 / 0 | Te0/x/0/17 | 0 / 0 | TF0/x/0/31 | 0 / 0 | Te0/x/2/2 | 0 / 1 |
| Te0/x/0/4 | 0 / 0 | Te0/x/0/18 | 0 / 0 | TF0/x/0/32 | 0 / 1 | Te0/x/2/3 | 0 / 0 |
| Te0/x/0/5 | 0 / 0 | Te0/x/0/19 | 0 / 0 | TF0/x/0/33 | 0 / 1 | Te0/x/2/4 | 0 / 1 |
| Te0/x/0/6 | 0 / 0 | Te0/x/0/20 | 0 / 1 | TF0/x/0/34 | 0 / 1 | Te0/x/2/5 | 0 / 0 |
| Te0/x/0/7 | 0 / 0 | Te0/x/0/21 | 0 / 1 | TF0/x/0/35 | 0 / 1 | Te0/x/2/6 | 0 / 1 |
| Te0/x/0/8 | 0 / 1 | Te0/x/0/22 | 0 / 1 | TF0/x/0/36 | 0 / 0 | Te0/x/2/7 | 0 / 0 |
| Te0/x/0/9 | 0 / 1 | Te0/x/0/23 | 0 / 1 | TF0/x/0/37 | 0 / 0 | Te0/x/2/8 | 0 / 0 |
| Te0/x/0/10 | 0 / 1 | TF0/x/0/24 | 0 / 1 | TF0/x/0/38 | 0 / 0 | Te0/x/2/9 | 0 / 1 |
| Te0/x/0/11 | 0 / 1 | TF0/x/0/25 | 0 / 1 | TF0/x/0/39 | 0 / 0 | Te0/x/2/10 | 0 / 0 |
| Te0/x/0/12 | 0 / 1 | TF0/x/0/26 | 0 / 1 | Hu0/x/1/0 | 0 / 0 | Te0/x/2/11 | 0 / 1 |
| Te0/x/0/13 | 0 / 1 | TF0/x/0/27 | 0 / 1 | Hu0/x/1/1 | 0 / 1 | - | - |



### NCS55-36X100G-A-SE

Finally, this line card is using 4 Jericho+ with new generation eTCAM.

![NC55-36X100G-A-SE.jpg]({{site.baseurl}}/images/NC55-36X100G-A-SE.jpg)

| Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core | Interface | NPU/Core |
|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|:-----:|
| Hu0/x/0/0 | 0 / 1 | Hu0/x/0/9 | 1 / 1  | Hu0/x/0/18 | 2 / 1 | Hu0/x/0/27 | 3 / 1 |
| Hu0/x/0/1 | 0 / 1 | Hu0/x/0/10 | 1 / 1 | Hu0/x/0/19 | 2 / 1 | Hu0/x/0/28 | 3 / 1 |
| Hu0/x/0/2 | 0 / 1 | Hu0/x/0/11 | 1 / 1 | Hu0/x/0/20 | 2 / 1 | Hu0/x/0/29 | 3 / 1 |
| Hu0/x/0/3 | 0 / 0 | Hu0/x/0/12 | 1 / 0 | Hu0/x/0/21 | 2 / 0 | Hu0/x/0/30 | 3 / 0 |
| Hu0/x/0/4 | 0 / 0 | Hu0/x/0/13 | 1 / 0 | Hu0/x/0/22 | 2 / 0 | Hu0/x/0/31 | 3 / 0 |
| Hu0/x/0/5 | 0 / 0 | Hu0/x/0/14 | 1 / 0 | Hu0/x/0/23 | 2 / 0 | Hu0/x/0/32 | 3 / 0 |
| Hu0/x/0/6 | 0 / 0 | Hu0/x/0/15 | 1 / 0 | Hu0/x/0/24 | 2 / 0 | Hu0/x/0/33 | 3 / 0 |
| Hu0/x/0/7 | 0 / 1 | Hu0/x/0/16 | 1 / 1 | Hu0/x/0/25 | 2 / 1 | Hu0/x/0/34 | 3 / 1 |
| Hu0/x/0/8 | 0 / 0 | Hu0/x/0/17 | 1 / 0 | Hu0/x/0/26 | 2 / 0 | Hu0/x/0/35 | 3 / 0 |
