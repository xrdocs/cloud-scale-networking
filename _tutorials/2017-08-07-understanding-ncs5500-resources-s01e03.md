---
published: true
date: '2017-08-07 13:46 +0200'
title: Understanding NCS5500 Resources (S01E03)
author: Nicolas Fevrier
excerpt: Third post on the NCS5500 Resources focusing on IPv6 prefixes
position: top
tags:
  - ncs5500
  - ncs 5500
  - lpm
  - lem
  - routes
  - prefixes
  - eTCAM
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %} 

## S01E03 IPv6 Prefixes

### Previously on "Understanding NCS5500 Resources"

In the previous posts, we introduced the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/) and we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/).

All the principles described below and the examples used to illustrate them were validated in August 2017 with Jericho-based systems, using scale (with eTCAM) and base (without eTCAM) line cards and running the two IOS XR releases available: 6.1.4 and 6.2.2.
{: .notice--info}

### IPv6 routes and FIB Profiles

Please take a few minutes to read the [S01E02](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/) to understand the different databases used to store routes in NCS5500:

![Resources]({{site.baseurl}}/images/resources.jpg){: .align-center}

- **LPM**: Longest Prefix Match Database (or KAPS) is an SRAM used to store IPv4 and IPv6 prefixes. 
- **LEM**: Large Exact Match Database also used to store specific IPv4 and IPv6 routes, plus MAC addresses and MPLS labels.
- **eTCAM**: external TCAMs, only present in the -SE "scale" line cards and systems. As the name implies, they are not a resource inside the Forwarding ASIC, it's an additional memory used to extend unicast route and ACL / classifiers scale.

We explained how the different profiles influenced the prefixes storing in different databases for base and scale systems or line cards. The principles for IPv6 are similar but things are actually simpler: the order of operation will be exactly the same, regardless of the FIB profile activated and regardless of the type of line card (**base** or **scale**).

![IPv6-order.jpg]({{site.baseurl}}/images/IPv6-order.jpg){: .align-center}

The logic behind this decision: IPv6/48 prefixes are by far the largest population of the public table.

![ipv6-table.jpg]({{site.baseurl}}/images/ipv6-table.jpg){: .align-center}
(From [BGPv6 table on Twitter](https://twitter.com/bgp6_table))

To avoid any misunderstanding, let's review the IPv6 resource allocation / distribution for each profile and line card type quickly, starting with the **Base systems** with **Host-optimized** FIB profile:

![IPv6-base-host-distr.jpg]({{site.baseurl}}/images/IPv6-base-host-distr.jpg){: .align-center}

**Base systems** with **Internet-optimized** FIB profile:

![IPv6-base-internet-distr.jpg]({{site.baseurl}}/images/IPv6-base-internet-distr.jpg){: .align-center}

**Scale systems** regardless of FIB profile:

![IPv6-scale-distribution.jpg]({{site.baseurl}}/images/IPv6-scale-distribution.jpg){: .align-center}

See ? Pretty easy. By default, IPv6/48 are moved into LEM and the all other IPv6 prefixes are pushed into LPM.

### Lab verification

LPM is an algorithmic memory. That means, the capacity will depend on the prefix distribution and how many have been programmed at a given moment. We will use a couple of examples below to illustrate below how the routes are moved but you should not rely on the "estimated capacity" to based your capacity planning. Only a real internet table will give you a reliable idea of the available space.

In slot 0/0, we have a **base line card** (18H18F) using an Internet-optimized profile. In slot 0/6, we use a **scale line card** (24H12F-SE).

Also, keep in mind we are announcing ordered prefixes which are fine in a lab context to verify where the system will store the routes but it's not a realistic scenario (compared to a real internet table for instance).

__IPv6/48 Routes__

IPv6/48 prefixes are stored in LEM:

![IPv6-48.jpg]({{site.baseurl}}/images/IPv6-48.jpg){: .align-center}

First we advertise 20,000 IPv6/48 routes and check the different databases.
On **Base line cards**:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /48 | utility wc -l
<mark>20000</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lem location 0/0/CPU0

HW Resource Information
    Name                            : lem

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 20107    (3 %)
        iproute                     : 5        (0 %)
        ip6route                    : <mark>20000</mark>    (3 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 117926
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 172      (0 %)
        iproute                     : 29       (0 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

Note: for readability, we will only display NPU-0 information. In the full output of the show command, we will have from NPU-0 to NPU-0 on 18H18F and from NPU-0 to NPU-3 on the 24H12F-SE.
{: .notice--info}

On **Scale line cards**:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lem location 0/6/CPU0

HW Resource Information
    Name                            : lem

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green


Current Usage
    NPU-0
        Total In-Use                : 20127    (3 %)
        iproute                     : 24       (0 %)
        ip6route                    : <mark>20000</mark>    (3 %)
        mplslabel                   : 102      (0 %)


RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 118638
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 144      (0 %)
        iproute                     : 0        (0 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

With 20,000 IPv6/48 prefixes, as expected, it's only 3% of the 786,432 entries of LEM.

Just for verification, we will advertise 200,000 then 400,000 IPv6/48 routes. And of course the LEM estimated max entries will stay constant. LEM is very different than LPM from this perspective.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /48 | utility wc -l
200000
RP/0/RP0/CPU0:NCS5508-1-614#
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lem location 0/0/CPU0  | i "(Estim|In-Use)"
        Estimated Max Entries       : 786432
        Total In-Use                : <mark>200107   (25 %)</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 117926
        Total In-Use                : 172      (0 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lem location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 786432
        Total In-Use                : <mark>200127   (25 %)</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 118638
        Total In-Use                : 144      (0 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /48 | utility wc -l
<mark>400000</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lem location 0/0/CPU0  | i "(Estim|In-Use)"
        Estimated Max Entries       : 786432
        Total In-Use                : <mark>400107   (51 %)</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 117926
        Total In-Use                : 172      (0 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lem location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 786432
        Total In-Use                : <mark>400127   (51 %)</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 118638
        Total In-Use                : 144      (0 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

__Non IPv6/48 Routes ?__

From IPv6/1 to IPv6/47 and from IPv6/49 to IPv6/128, all these prefixes will be stored in LPM.

![IPv6-47-0.jpg]({{site.baseurl}}/images/IPv6-47-0.jpg){: .align-center}

![IPv6-128-49.jpg]({{site.baseurl}}/images/IPv6-128-49.jpg){: .align-center}

The estimated max prefixes will be very different for each test and will also differ depending on the number of routes we advertise.

__IPv6/32 Routes__

Let's see is the occupation for 20,000 / 40,000 and 60,000 IPv6/32 prefixes.

On **base line cards** with 20,000 IPv6/32:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /32 | utility wc -l
<mark>20000</mark>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0
HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 493046
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 20172    (4 %)
        iproute                     : 29       (0 %)
        ip6route                    : <mark>20117    (4 %)</mark>
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

On **scale line cards** with IPv6/32:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0
HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 492362
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 20144    (4 %)
        iproute                     : 0        (0 %)
        ip6route                    : <mark>20117    (4 %)</mark>
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

40,000 IPv6/32 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 477274
        Total In-Use                : 40172    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 478572
        Total In-Use                : 40144    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

60,000 IPv6/32 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 459800
        Total In-Use                : 60172    (13 %)
RP/0/RP0/CPU0:NCS5508-1-614#
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 460686
        Total In-Use                : 60144    (13 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

__IPv6/56 Routes__

20,000 IPv6/56 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /56 | utility wc -l
20000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 239664
        Total In-Use                : 20172    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 489198
        Total In-Use                : 20144    (4 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

40,000 IPv6/56 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /56 | utility wc -l
40000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 220600
        Total In-Use                : 40172    (18 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 475320
        Total In-Use                : 40144    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

60,000 IPv6/56 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /56 | utility wc -l
60000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 201192
        Total In-Use                : 60172    (30 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 458492
        Total In-Use                : 60144    (13 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

__IPv6/64 Routes__

20,000 IPv6/64 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /64 | utility wc -l
20000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 239664
        Total In-Use                : 20172    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 489198
        Total In-Use                : 20144    (4 %)
RP/0/RP0/CPU0:NCS5508-1-614#        
</code>
</pre>
</div>

40,000 IPv6/64 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /64 | utility wc -l
40000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 220600
        Total In-Use                : 40172    (18 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 475320
        Total In-Use                : 40144    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

60,000 IPv6/64 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /64 | utility wc -l
60000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 201192
        Total In-Use                : 60172    (30 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 458492
        Total In-Use                : 60144    (13 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

__IPv6/128 Routes__

20,000 IPv6/128 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /128 | utility wc -l
20000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 238848
        Total In-Use                : 20172    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 239330
        Total In-Use                : 20144    (8 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

40,000 IPv6/128 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /128 | utility wc -l
40000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 220186
        Total In-Use                : 40172    (18 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 220446
        Total In-Use                : 40144    (18 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

60,000 IPv6/128 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /128 | utility wc -l
60000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 200914
        Total In-Use                : 60172    (30 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 201098
        Total In-Use                : 60144    (30 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

80,000 IPv6/128 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh route ipv6 bgp | i /128 | utility wc -l
60000
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 181075
        Total In-Use                : 80173    (44 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 181219
        Total In-Use                : 80145    (44 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

100,000 IPv6/128 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 161334
        Total In-Use                : 100172   (62 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 161456
        Total In-Use                : 100144   (62 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

120,000 IPv6/128 prefixes:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/0/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 141370
        Total In-Use                : 120172   (85 %)
RP/0/RP0/CPU0:NCS5508-1-614#sh contr npu resources lpm location 0/6/CPU0 | i "(Estim|In-Use)"
        Estimated Max Entries       : 141476
        Total In-Use                : 120144   (85 %)
RP/0/RP0/CPU0:NCS5508-1-614#
</code>
</pre>
</div>

| Pfx | Base Max Pfx | Scale Max Pfx |
|------|------|-------|
| 20k IPv6/32 | LPM: 489,903 | LPM: 492,387 |
| 40k IPv6/32 | LPM: 475,663 | LPM: 478,583 |
| 60k IPv6/32 | LPM: 458,713 | LPM: 460,693 |
| 80k IPv6/32 | LPM: 440,257 | LPM: 440,929 |
| 100k IPv6/32 | LPM: 421,187 | LPM: 421,733 |
| 200k IPv6/32 | LPM: 322,395 | LPM: 323,017 |
| 250k IPv6/32 | LPM: 272,903 | LPM: 273,141 |
| 20k IPv6/48 | LEM: 786,432 | LEM: 786,432 |
| 40k IPv6/48 | LEM: 786,432 | LEM: 786,432 |
| 60k IPv6/48 | LEM: 786,432 | LEM: 786,432 |
| 80k IPv6/48 | LEM: 786,432 | LEM: 786,432 |
| 100k IPv6/48 | LEM: 786,432 | LEM: 786,432 |
| 200k IPv6/48 | LEM: 786,432 | LEM: 786,432 |
| 20k IPv6/56 | LPM: 486,773 | LPM: 489,223 |
| 40k IPv6/56 | LPM: 474,051 | LPM: 475,331 |
| 60k IPv6/56 | LPM: 457,623 | LPM: 458,501 |
| 80k IPv6/56 | LPM: 439,433 | LPM: 440,103 |
| 100k IPv6/56 | LPM: 420,525 | LPM: 421,069 |
| 200k IPv6/56 | LPM: 322,061 | LPM: 322,349 |
| 250k IPv6/56 | LPM: 272,637 | LPM: 272,873 |
| 20k IPv6/64 | LPM: 239,675 | LPM: 489,223 |
| 40k IPv6/64 | LPM: 220,605 | LPM: 475,331 |
| 60k IPv6/64 | LPM: 201,195 | LPM: 458,501 |
| 80k IPv6/64 | LPM: 181,283 | LPM: 440,103 |
| 100k IPv6/64 | LPM: 161,503 | LPM: 421,069 |
| 120k IPv6/64 | LPM: 141,511 | LPM: 401,163 |
| 20k IPv6/128 | LPM: 238,848 | LPM: 239,330 |
| 40k IPv6/128 | LPM: 220,186 | LPM: 220,446 |
| 60k IPv6/128 | LPM: 200,914 | LPM: 201,098 |
| 80k IPv6/128 | LPM: 181,075 | LPM: 181,219 |
| 100k IPv6/128 | LPM: 161,334 | LPM: 161,456 |
| 120k IPv6/128 | LPM: 141,370 | LPM: 141,476 |

Again this chart is just provided for information with "aligned"/"sorted" routes, not really representing a real internet distribution. Take a look at the [former post for a production router with public view IPv4+IPv6](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/).

In next posts, we will cover Encapsulation database, FEC and ECMP FEC database, MPLS use-cases and the classifiers/ACLs. Stay tuned.