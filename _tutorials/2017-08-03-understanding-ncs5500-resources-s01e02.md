---
published: true
date: '2017-08-03 17:41 +0200'
title: Understanding NCS5500 Resources (S01E02)
author: Nicolas Fevrier
excerpt: Second post on the NCS5500 Resources focusing on IPv4 prefixes
tags:
  - NCS5500
  - NCS 5500
  - LPM
  - LEM
  - eTCAM
  - XR
  - IOSXR
  - Memory
position: top
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %} 

## S01E02 IPv4 Prefixes

### Previously on "Understanding NCS5500 Resources"

In the [previous post](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/), we introduced the different routers and line cards in NCS5500 portfolio. We classified them into two categories: with or without external TCAM (eTCAM). And we introduced the different databases available to store information, inside and outside the Forwarding ASIC (FA).

All the principles described below and the examples used to illustrate them were validated in August 2017 with Jericho-based systems, using scale (with eTCAM) and base (without eTCAM) line cards and running the two IOS XR releases available: 6.1.4 and 6.2.2. Jericho+ based systems will be used in a follow up post in the same series (season 2 ;)
{: .notice--info}

### IPv4 routes and FIB Profiles

A quick refresh will be very useful to understand how routes are stored in NCS5500:

![Resources]({{site.baseurl}}/images/resources.jpg){: .align-center}

- **LPM**: Longest Prefix Match Database (sometimes referred to as KAPS for KBP Assisted Prefix Search, KBP being itself Knowledge Based Processor) is an SRAM used to store IPv4 and IPv6 prefixes. Scale: variable from 128k to 400k entries. We can perform variable length prefix lookup in LPM.
- **LEM**: Large Exact Match Database also used to store specific IPv4 and IPv6 routes, plus MAC addresses and MPLS labels. Scale: 786k entries. We perform exact match lookup in LEM.
- **eTCAM**: external TCAMs, only present in the -SE "scale" line cards and systems. As the name implies, they are not a resource inside the Forwarding ASIC, it's an additional memory used to extend unicast route and ACL / classifiers scale. Scale: 2M IPv4 entries. We can also perform variable length prefix lookup in eTCAM.

The origin of the prefixes is not relevant. They can be received from OSPF, ISIS, BGP but also static routes. It doesn't influence which database will be used to store them. Only the address-family (IPv4 in this discussion) and the subnet length of the prefix will be used in the decision process.

Hardware programming is done through an abstraction layer: Data-Plane Agent (DPA)

![DPA]({{site.baseurl}}/images/DPA.jpg){: .align-center}

Also, it's important to remember we are not talking about BGP paths here but about FIB entries: if we have 10 internet transit providers advertising more or less the same 700k-ish routes (with 10 next-hop addresses), we don't have 7M entries in the FIB but 700k. Few exceptions exist (like using different VRFs for each transit provider) but they are out of the scope of this post.

Originally, IPv4/32 are going in LEM and all other prefix lengths (IPv4/31-/0) are stored in LPM.
We changed this default behavior by implementing FIB profiles: *Host-optimized* or *Internet-Optimized*.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500(config)#hw-module fib ipv4 scale ?
  <mark>host-optimized-disable</mark>  Configure Host optimization by default
  <mark>internet-optimized</mark>      Configure Intetrnet optimized
RP/0/RP0/CPU0:NCS5500(config)#
</code>
</pre>
</div>

Host-optimized is the default option. Committing a change in the configuration will prompt you to reload the line-cards or chassis to enable the new profile.
{: .notice--info}

For a base line card (those without -SE in the product ID), we will have the following order of operation:

![IPv4 Host Optimized Order]({{site.baseurl}}/images/Host-Optimized-IPv4.jpg){: .align-center}

When a packet is received, the FA performs a lookup on the destination address:
- first lookup is performed in the LEM searching for a IPv4/32 exact match
- second lookup is accessing the LPM searching for a variable length match between IPv4/31 and IPv4/25
- third lookup is done in the LEM again, searching for a IPv4/24 exact match
- finally, the fourth lookup is checking the LPM a second time searching for a variable length match between IPv4/23 and /0

All is done in one single clock tick, it doesn't require any kind of recirculation and doesn't impact the performance (in bandwidth or in packet per second).

![IPv4 Host-optimized Allocation]({{site.baseurl}}/images/non-eTCAM-IPv4-HostOpt--.jpg){: .align-center}

This mode is particularly useful with a large number of IPv4/32 and IPv4/24 in the routing table. It could be the case for hosting companies or data centers.

Using the configuration above, you can decide to enable the **Internet-optimized** mode. This is a feature activated globally and not per line card. After reload, you will see a very different order of operation and prefix distribution in the various databases with base line cards and systems:

![IPv4 Internet Optimized Order]({{site.baseurl}}/images/non-SE-Int-Optimized-IPv4.jpg){: .align-center}

The order of operation LEM/LPM/LEM/LPM is now replaced by an LPM/LEM/LEM/LPM approach.
- first lookup is in LPM searching for a match between IPv4/32 and IPv4/25
- second lookup is performed in LEM for an exact match on IPv4/24 and IPv4/23.
- third lookup is done in LEM too, and this time for also an exact match on IPv4/20
- fourth and final step, a variable length lookup is executed in LPM for everything between IPv4/22 and /0

Here again, everything is performed in one cycle and the activation of the Internet Optimized mode doesn't impact the forwarding performance.

![non-eTCAM-IPv4-IntOpt-.jpg]({{site.baseurl}}/images/non-eTCAM-IPv4-IntOpt-.jpg){: .align-center}

As the name implies, this profile has been optimized to move the largest route population present on the Internet (IPv4/24, IPv4/23, IPv4/20) into the largest memory database: the LEM. If you followed carefully, you noticed that a couple of improvement are needed to implement this sequence of lookup. 

First, the match on LEM needs to be on an exact prefix length but the step two is done on IPv4/24 and IPv4/23. Indeed a function in DPA splits all IPv4/23 received from the upper FIB process in two. Each IPv4/23 is programmed as two sub-sequent IPv4/24s in hardware. We will illustrate this case with an example in the lab later in this post, advertising 300,000 IPv4/23.

Second, the exact match in step 3 for IPv4/20 in LEM is only possible if we don't have any IPv4/22 or IPv4/21 prefixes overlapping with this IPv4/20. This implies the system performs another pro-active check to verify we don't have overlap. If an overlap happens, the IPv4/20 prefix is moved from LEM to LPM dynamically. We will illustrate this mechanism later in the post, advertising 100,000 IPv4/20 first, then advertising 100,000 IPv4/21 overlapping on the IPv4/20. We will see the IPv4/20 moved into LPM automatically.

With this Internet Optimized profile activated, it's possible to store a full internet view on base systems and line cards (we will present a couple of examples at the end of the documents).

- LEM is 786k large
- LPM scales from 256k to 350-400k (depending on the internet distribution, this algorithmic memory is dynamically optimized)
- Total IPv4 scale for base systems is 786k + 350k = 1,136k routes
{: .notice--info}

What about the scale line cards and routers (NCS5501-SE, NCS5502-SE and all the -SE line cards) ?

The two optimized profiles described earlier don't impact the lookup process on scale systems, which will always follow this order of operation:

![eTCAM IPv4 Order]({{site.baseurl}}/images/-SE-IPv4-order.jpg){: .align-center}

Just a two-step lookup here:
- first lookup is in LEM for an exact match on IPv4/32
- second and last lookup in the large eTCAM for everything between IPv4/31 and /0

Needless to say, it is all done in one single operation in the Forwarding ASIC.

![eTCAM-IPv4-.jpg]({{site.baseurl}}/images/eTCAM-IPv4-.jpg){: .align-center}

- LEM is 786k large
- eTCAM can offer up to 2M IPv4 entries
- Total IPv4 scale for scale systems is 786k + 2M = 2,786k routes
{: .notice--info}

### Lab verification

Let's try to illustrate it in the lab, injecting different types of IPv4 routes. 

On NCS5500, the IOS XR CLI to verify the resource utilization is "show controller npu resources all location 0/x/CPU0". 

We will advertise prefixes from a test device and check the memory utilization. It's not an ideal approach because, for the sake of simplicity, the routes advertised through BGP are contiguous:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500#sh route bgp

B    2.0.0.0/32 [20/0] via 192.168.1.2, 04:13:13
B    2.0.0.1/32 [20/0] via 192.168.1.2, 04:13:13
B    2.0.0.2/32 [20/0] via 192.168.1.2, 04:13:13
B    2.0.0.3/32 [20/0] via 192.168.1.2, 04:13:13
B    2.0.0.4/32 [20/0] via 192.168.1.2, 04:13:13
B    2.0.0.5/32 [20/0] via 192.168.1.2, 04:13:13
B    2.0.0.6/32 [20/0] via 192.168.1.2, 04:13:13
[...]

RP/0/RP0/CPU0:NCS5500#
</code>
</pre>
</div>

Despite common belief, it's not an ideal situation. On the contrary, algorithmic memories (like LPM) will be capable of much higher scale with real internet prefix-length distribution. Nevertheless, it's still an ok approach to demonstrate where the prefixes are stored (based on the subnet length).

We will take a look at two systems using scale line cards (24H12F) in slot 0/6 and base line cards (18H18F) in slot 0/0, and running two different IOS XR releases (6.1.4 and 6.2.2).
  
__200k IPv4 /32 routes__

Let's get started with the advertisement of 200,000 IPv4/32 prefixes.

On **base line cards** running **Host-optimized** profile, IPv4/32 routes are going to LEM.

![host-32.jpg]({{site.baseurl}}/images/host-32.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh route sum
Route Source                     Routes     Backup     Deleted     Memory(bytes)
connected                        6          1          0           1680
local                            7          0          0           1680
static                           2          0          0           480
ospf 100                         0          0          0           0
dagr                             0          0          0           0
bgp 100                          <mark>200000</mark>     0          0           48000000
Total                            200015     1          0           48003840

RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /32 | utility wc -l
<mark>200000</mark>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

HW Resource Information
    Name                            : lem

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
[...]

Current Usage
    NPU-0
        Total In-Use                : 200131   (25 %)
        iproute                     : <mark>200029</mark>   (25 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)
[...]

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 87036
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 148      (0 %)
        iproute                     : <mark>5</mark>        (0 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

Estimated Max Entries (and the Current Usage percentage derived from it) are only estimations provided by the Forwarding ASIC based on the current memory occupation and prefix distribution. It's not always linear and should  be taken with a grain of salt.
{: .notice--info}

On **base line cards** running **Internet-optimized** profile, IPv4/32 routes are going to LPM:

![internet-32.jpg]({{site.baseurl}}/images/internet-32.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /32 | utility wc -l
<mark>200000</mark>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

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
        Total In-Use                : 107      (0 %)
        iproute                     : <mark>5</mark>        (0 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 323057
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 200171   (62 %)
        iproute                     : <mark>200029</mark>   (62 %)
        ip6route                    : 116      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

Finaly, on **scale line card**, regardless of the profile enabled, the IPv4/32 are stored in LEM and not eTCAM:

![eTCAM-32.jpg]({{site.baseurl}}/images/eTCAM-32.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/6/CPU0

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
        Total In-Use                : 200127   (25 %)
        iproute                     : <mark>200024</mark>   (25 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources exttcamipv4 location 0/6/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 10       (0 %)
        iproute                     : <mark>10</mark>       (0 %)
        ipmcroute                   : 0        (0 %)


RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>
  
__500k IPv4 /24 routes__

In this second example, we announce 500,000 IPv4/24 prefixes.
With both **host-optimized** and **internet-optimized** profiles on base line cards, we will see these prefixes moved into LEM.

![host-24.jpg]({{site.baseurl}}/images/host-24.jpg){: .align-center}
![internet-24-23.jpg]({{site.baseurl}}/images/internet-24-23.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /24 | utility wc -l
<mark>500000</mark>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

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
        Total In-Use                : 500131   (64 %)
        iproute                     : <mark>500029</mark>   (64 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 87036
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 148      (0 %)
        iproute                     : <mark>5</mark>        (0 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

On **scale line cards**, only IPv4/32s are going to LEM. The rest (that includes our 500,000 IPv4/24s) will be pushed to the external TCAM:

![eTCAM-rest.jpg]({{site.baseurl}}/images/eTCAM-rest.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/6/CPU0

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
        Total In-Use                : 127      (0 %)
        iproute                     : <mark>24</mark>       (0 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/6/CPU0

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
        iproute                     : <mark>0</mark>        (0 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources exttcamipv4 location 0/6/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 500010   (24 %)
        iproute                     : <mark>500010</mark>   (24 %)
        ipmcroute                   : 0        (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>
  
__300k IPv4 /23 routes__

In this third example, we announce 300,000 IPv4/23 prefixes.
With the **Host-optimized** profiles on **base line cards**, they will be moved to the LPM.

![host-23-20.jpg]({{site.baseurl}}/images/host-23-20.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh route sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
connected                        6          1          0           1680
local                            7          0          0           1680
static                           2          0          0           480
ospf 100                         0          0          0           0
dagr                             0          0          0           0
bgp 100                          <mark>300000</mark>     0          0           72000000
Total                            300015     1          0           72003840

RP/0/RP0/CPU0:NCS5500-614#sh route bgp

B    110.0.0.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.2.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.4.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.6.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.8.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.10.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.12.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.14.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.16.0/23 [20/0] via 192.168.1.2, 00:18:17
B    110.0.18.0/23 [20/0] via 192.168.1.2, 00:18:17
^c
RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /23 | utility wc -l
<mark>300000</mark>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

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
        Total In-Use                : 131      (0 %)
        iproute                     : <mark>29</mark>       (0 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 261968
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Red
        OOR State Change Time       : 2017.Aug.04 06:59:54 UTC

Current Usage
    NPU-0
        Total In-Use                : 261714   (100 %)
        iproute                     : <mark>261571</mark>   (100 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

Only 261k IPv4/23 prefixes out of the 300k were programmed. Then, we reached the max of the memory capacity (we removed the error messages reporting that extra entries have not been programmed in hardware because the LPM capacity was exceeded).

Let's enable the **Internet-optimized** profile (and reload).

This time, the 300,000 IPv4/23 will be split into two, creating 600,000 IPv4/24. And we will move them into LEM.
Note: routes are not split into RIB/FIB but just when programmed into the hardware. That's why a show route will display 300,000 entries and the show contr npu resource will display 600,000.

![internet-24-23.jpg]({{site.baseurl}}/images/internet-24-23.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /23 | utility wc -l
<mark>300000</mark>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

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
        Total In-Use                : 600107   (76 %)
        iproute                     : <mark>600005</mark>   (76 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 140729
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 171      (0 %)
        iproute                     : <mark>29</mark>       (0 %)
        ip6route                    : 116      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

The same example with **scale line cards** will not be dependant on the optimized profile activated, all the IPv4/23 routes will be stored in external TCAM:

![eTCAM-rest.jpg]({{site.baseurl}}/images/eTCAM-rest.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/6/CPU0

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
        Total In-Use                : 127      (0 %)
        iproute                     : <mark>24</mark>       (0 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/6/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 117819
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 143      (0 %)
        iproute                     : <mark>0</mark>        (0 %)
        ip6route                    : 116      (0 %)
        ipmcroute                   : 50       (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources exttcamipv4 location 0/6/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 300010   (15 %)
        iproute                     : <mark>300010</mark>   (15 %)
        ipmcroute                   : 0        (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>
  
__100k IPv4 /20 routes__

In this last example, we announce 100,000 IPv4/20 prefixes.
You got it, so no need to describe:
- the **host-optimized** profile on **base line cards** where these 100k will be stored in LPM 
- the **scale cards** where these routes will be pushed to the external TCAM

Let's focus on the behavior with **base line cards** running an **Internet-optimized** profile.
We only advertise IPv4/20 routes and no overlapping routes, they will be all stored in LEM.

![internet-20-no-overlap.jpg]({{site.baseurl}}/images/internet-20-no-overlap.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh bgp sum
[...]
Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
10.11.12.1        0   100   13287    1466  4300005    0    0 13:11:23     100000

RP/0/RP0/CPU0:NCS5500-614#sh route bgp

B    3.0.0.0/20 [200/0] via 192.168.1.2, 10:05:09
B    3.0.16.0/20 [200/0] via 192.168.1.2, 10:05:09
B    3.0.32.0/20 [200/0] via 192.168.1.2, 10:05:09
B    3.0.48.0/20 [200/0] via 192.168.1.2, 10:05:09
B    3.0.64.0/20 [200/0] via 192.168.1.2, 10:05:09
B    3.0.80.0/20 [200/0] via 192.168.1.2, 10:05:09
B    3.0.96.0/20 [200/0] via 192.168.1.2, 10:05:09
[...]

RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /20 | utility wc -l
100000
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

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
        Total In-Use                : 100002   (13 %)
        iproute                     : <mark>100002</mark>   (13 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 0        (0 %)
        
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 148883
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
        
Current Usage
    NPU-0
        Total In-Use                : 153      (0 %)
        iproute                     : <mark>19</mark>       (0 %)
        ip6route                    : 113      (0 %)
        ipmcroute                   : 0        (0 %)
        
RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

Now, we advertise 100,000 new IPv4/21 routes. All are overlapping the IPv4/20 we announced earlier. The IPv4/20 will no longer be stored in LEM but will be moved into LPM, for a total of 200,000 entries:

![internet-20-overlap.jpg]({{site.baseurl}}/images/internet-20-overlap.jpg){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500-614#sh bgp sum

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
10.11.12.1        0   100   13901    1474  4600005    0    0 13:19:24     200000

RP/0/RP0/CPU0:NCS5500-614#sh route bgp
Sat Aug  5 00:15:38.987 UTC

B    3.0.0.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.0.0/21 [200/0] via 192.168.1.2, 00:02:27
B    3.0.16.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.16.0/21 [200/0] via 192.168.1.2, 00:02:27
B    3.0.32.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.32.0/21 [200/0] via 192.168.1.2, 00:02:27
B    3.0.48.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.48.0/21 [200/0] via 192.168.1.2, 00:02:27
B    3.0.64.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.64.0/21 [200/0] via 192.168.1.2, 00:02:27
B    3.0.80.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.80.0/21 [200/0] via 192.168.1.2, 00:02:27
B    3.0.96.0/20 [200/0] via 192.168.1.2, 00:00:02
B    3.0.96.0/21 [200/0] via 192.168.1.2, 00:02:27
[...]

RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /20 | utility wc -l
100000
RP/0/RP0/CPU0:NCS5500-614#sh route bgp | i /21 | utility wc -l
100000
RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lem location 0/0/CPU0

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
        Total In-Use                : 2        (0 %)
        iproute                     : <mark>2</mark>        (0 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 0        (0 %)

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/0/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 539335
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 200153   (37 %)
        iproute                     : <mark>200019</mark>   (37 %)
        ip6route                    : 113      (0 %)
        ipmcroute                   : 0        (0 %)

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>


### Real use-cases

To conclude, let's illustrate with real but anonymized use-cases.

On a **base system** running IOS XR 6.2.2 with **internet-optimized** profile and a “small” internet table.

<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RP0/CPU0:5501#show route sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
connected                        2          3          0           1200         
local                            5          0          0           1200         
local LSPV                       1          0          0           240          
static                           2          0          0           480          
ospf 1                           677        2          0           163072       
bgp xxxx                         <mark>615680</mark>     10         0           147765600    
dagr                             0          0          0           0            
Total                            616367     15         0           147931792    

RP/0/RP0/CPU0:5501#show dpa resources iproute location 0/0/CPU0

"iproute" DPA Table (Id: 17, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution 
Prefix   Actual       Prefix   Actual
 /0       1            /1       0           
 /2       0            /3       0           
 /4       1            /5       0           
 /6       0            /7       0           
 /8       15           /9       9           
 /10      35           /11      102         
 /12      277          /13      527         
 /14      955          /15      1703        
 /16      12966        /17      7325        
 /18      12874        /19      23469       
 /20      35743        /21      39283       
 /22      72797        /23      60852       
 /24      346773       /25      3           
 /26      19           /27      21          
 /28      17           /29      13          
 /30      229          /31      0           
 /32      368         
[...]

RP/0/RP0/CPU0:5501#show contr npu resources all location 0/0/CPU0
 
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
        Total In-Use                : 498080   (63 %)
        iproute                     : <mark>507304</mark>   (65 %)
        ip6route                    : 12818    (2 %)
        mplslabel                   : 677      (0 %)

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 510070  
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : 192543   (38 %)
        iproute                     : <mark>176254</mark>   (35 %)
        ip6route                    : 15583    (3 %)
        ipmcroute                   : 0        (0 %)

</code>
</pre>
</div>

Same route distribution on a **scale system** running IOS XR 6.2.2:

<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RP0/CPU0:5501-SE#show route sum 
Route Source                     Routes     Backup     Deleted     Memory(bytes)
connected                        4          3          0           1680         
local                            7          0          0           1680         
local LSPV                       1          0          0           240          
static                           2          0          0           480          
ospf 1                           677        2          0           163072       
bgp xxxx                         <mark>615681</mark>     10         0           147765840    
dagr                             0          0          0           0            
Total                            616372     15         0           147932992    
 
RP/0/RP0/CPU0:5501-SE#show dpa resources iproute location 0/0/CPU0
 
"iproute" DPA Table (Id: 17, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution 
Prefix   Actual            Capacity    Prefix   Actual            Capacity 
 /0       1                 20           /1       0                 20          
 /2       0                 20           /3       0                 20          
 /4       1                 20           /5       0                 20          
 /6       0                 20           /7       0                 20          
 /8       15                20           /9       9                 20          
 /10      35                205          /11      102               409         
 /12      277               818          /13      527               1636        
 /14      955               3275         /15      1703              5731        
 /16      12966             42368        /17      7325              25379       
 /18      12874             42571        /19      23469             86576       
 /20      35743             127308       /21      39283             141634      
 /22      72797             231894       /23      60852             207107      
 /24      346773            1105235      /25      4                 4298        
 /26      19                4503         /27      21                3275        
 /28      17                2865         /29      13                6959        
 /30      231               2865         /31      0                 205         
 /32      376               20          
 
[…]

RP/0/RP0/CPU0:5501-SE#show contr npu resources all location 0/0/CPU0
 
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
        Total In-Use                : 13887    (2 %)
        iproute                     : <mark>376</mark>      (0 %)
        ip6route                    : 12827    (2 %)
        mplslabel                   : 677      (0 %)
 
HW Resource Information
    Name                            : lpm
 
OOR Information
    NPU-0
        Estimated Max Entries       : 551346  
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
 
Current Usage
    NPU-0
        Total In-Use                : 15612    (3 %)
        iproute                     : <mark>0</mark>        (0 %)
        ip6route                    : 15589    (3 %)
        ipmcroute                   : 0        (0 %)
 
HW Resource Information
    Name                            : ext_tcam_ipv4
 
OOR Information
    NPU-0
        Estimated Max Entries       : 2048000 
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
 
Current Usage
    NPU-0
        Total In-Use                : 616012   (30 %)
        iproute                     : <mark>616012</mark>   (30 %)
        ipmcroute                   : 0        (0 %)
 
</code>
</pre>
</div>

Examples above show it's possible to store a full internet view in a **base system**. 

With a relatively small table of 616k routes, we have LEM used at approximatively 65%. But it's frequent to see larger internet tables (closer to 700k in August 2017), with many peering routes and internal routes. It still fits in but doesn't give much room for future growth. 

We advise to prefer **scale line cards and systems** for such use-cases.

In the next episode, we will cover IPv6 prefixes. Stay tuned.
