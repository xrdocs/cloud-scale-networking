---
published: false
date: '2017-08-03 17:41 +0200'
title: Understanding NCS5500 Resources (S01E02)
author: Nicolas Fevrier
excerpt: Second post on the NCS5500 Resource focusing on IPv4 prefixes
tags:
  - NCS5500
  - NCS 5500
  - LPM
  - LEM
  - eTCAM
  - XR
  - IOSXR
  - Memory
position: hidden
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %} 

## S01E02 IPv4 Prefixes

### Previously in "Understanding NCS5500 Resources"

In the [previous post](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/), we introduced the different routers and line cards in NCS5500 portfolio. We classified them in two categories: with or without external TCAM (eTCAM). And we introduced the different databases available to store information, inside and outside the Forwarding ASIC (FA).

All the principles described below and the examples used to illustrate them valid in August 2017 with Jericho-based systems, using scale (with eTCAM) and base (without eTCAM) line cards and running the two IOS XR releases available: 6.1.4 and 6.2.2. Jericho+ based systems will be used in a follow up post in the same series (season 2 ;)
{: .notice--info}

### IPv4 routes and FIB Profiles

A quick refresh will be very useful to understand how routes are stored in NCS5500:

![Resources]({{site.baseurl}}/images/resources.jpg){: .align-center}

- **LPM**: Longest Prefix Match Database (sometimes referred as KAPS for KBP Assisted Prefix Search, KBP being itself Knowledge Based Processor) is an SRAM used to store IPv4 and IPv6 prefixes. Scale: variable from 128k to 400k entries. We can perform variable length prefix lookup in LPM.
- **LEM**: Large Exact Match Database also used to store specific IPv4 and IPv6 routes, plus MAC addresses and MPLS labels. Scale: 786k entries. We perform exact match lookup in LEM.
- **eTCAM**: external TCAMs, only present in the -SE "scale" line cards and systems. As the name implies, they are not a resource inside the Forwarding ASIC, it's an additional memory used to extend unicast route and ACL / classifiers scale. Scale: 2M IPv4 entries. We can also perform variable length prefix lookup in eTCAM.

The origin of the prefix is not important. They can be received from OSPF, ISIS, BGP but also static routes. It doesn't impact which database will be used to store them.
Also, it's important to remind that we are not talking about BGP Paths here but about FIB entries: if we have 10 internet up-stream providers advertising the same 700k-ish routes (with 10 next-hop addresses), we don't have 7M entries in the FIB but 700k. Few exceptions exist (like using different VRFs for each transit provider) but they are out of the scope of this post.

Originally, IPv4/32 are going in LEM and all other prefix length (/31-/0) will be stored in LPM.
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
- first lookup is performed the LEM searching for a /32 exact match
- second lookup is accessing the LPM searching for a variable length match between /31 and /25
- third lookup is done the LEM again searching for a /24 exact match
- finally, the fourth lookup is checking the LPM a second time searching for a variable length match between /23 and /0
All is done in one single clock tick, it doesn't require any kind of recirculation and doesn't impact the performance (in bandwidth or in packet per second).

![IPv4 Host Opt Allocation]({{site.baseurl}}/images/non-eTCAM-IPv4-HostOpt-.jpg){: .align-center}

This mode is particularly useful if you are using a routing table with a large number of v4/32 and v4/24. It's the case for hosting company or if you are using the NCS5500 in a data center.

Using the configuration shown above, you can decide to enable the Internet-optimized mode. This is a feature activated on a chassis globally and not per line card. After reload, you will see a very different order of operation and prefix distribution in the various databases in your scale line cards and systems:

![IPv4 Internet Optimized Order]({{site.baseurl}}/images/non-SE-Int-Optimized-IPv4.jpg){: .align-center}

The order of operation LEM/LPM/LEM/LPM is now replaced by a LPM/LEM/LEM/LPM approach.
- first lookup is in LPM searching for a match between /32 and /25
- second lookup is performed in LEM for an exact match on /24. But a pro-active process already split all the /23 received from the upper FIB process. Each /23 is presented as two sub-sequent /24s.
- third lookup is done in LEM too and this time for also an exact match on /20. This implies the system performed another pro-active check to verify that we don't have any /22 or /21 prefixes overlapping with this /20. If it was the case, the /20 prefix is moved in the LPM and will be matched during the last lookup step.
- fourth and final step, a variable length lookup is executed in LPM for everything between /22 and /0
Here again, everything is performed in one cycle and the activation of the Internet Optimized mode doesn't impact the forwarding performance.

![non-eTCAM-IPv4-IntOpt.jpg]({{site.baseurl}}/images/non-eTCAM-IPv4-IntOpt.jpg){: .align-center}

As the name implies, this profile has been optimized to store the largest route population (v4/24, v4/23, v4/20) in the largest memory database: the LPM. And we introduced specific improvement and pre-processing to handle the v4/23 and v4/20 optimally. 
With this Internet Optimized profile activated, it's possible to store a full internet view on base systems and line cards (we will present a couple of examples at the end of the documents).

LEM is 786k large
LPM scales from 256k to 350-400k (depending on the internet distribution, this algorithmic memory is dynamically optimized)
Total IPv4 scale for base systems is 786k + 350k = 1,136k routes
{: .notice--info}

What about the scale line cards and routers (NCS5501-SE, NCS5502-SE and all the -SE line cards) ?

Former profiles don't impact the lookup process on scale systems, which will always follow this order of operation:

![eTCAM IPv4 Order]({{site.baseurl}}/images/-SE-IPv4-order.jpg){: .align-center}

Just a two-step lookup here:
- first lookup is in LEM for an exact match on /32
- second and last lookup in the large eTCAM for everything between /31 and /0
Needless to say, all done in one operation in the Forwarding ASIC.

![eTCAM-IPv4.jpg]({{site.baseurl}}/images/eTCAM-IPv4.jpg){: .align-center}

LEM is 786k large
eTCAM can offer up to 2M IPv4 entries
Total IPv4 scale for scale systems is 786k + 2M = 2,786k routes
{: .notice--info}

### Lab verification

Nothing better than an example and some show commands to illustrate it. On NCS5500, the IOS XR CLI to verify the resource utilization is "show controller npu resources all location 0/x/CPU0". In the lab, we will inject prefixes and check the memory utilization. It's not an ideal approach because, for the sake of simplicity, we advertise through BGP very ordered routes:

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

Despite common belief, it's not an ideal situation. On the contrary, algorithmic memories (like LPM) will show much higher scale with real internet prefix-length distribution.

Nevertheless, it's still an ok approach to demonstrate where the prefixes are stored based on the subnet length.
We will take a look at two systems using scale line cards (24H12F) in slot 0/6 and base line cards (18H18F) in slot 0/0, and running two different IOS XR releases (6.1.4 and 6.2.2). We will only specify it, if the output is different between the two releases.

__200k IPv4 /32 routes__

Let's get started with the advertisement of 200,000 IPv4/32 prefixes.

On base line cards running **Host-optimized** profile, /32 routes are going to LEM 

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

[…]

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

Estimated Max Entries (and the Current Usage percentage derived from it) are only estimation provided by the Forwarding ASIC based on the current prefix-length distribution. It's not always linear and should always be taken with a grain of salt.
{: .notice--info}

On base line cards running **Internet-optimized** profile, /32 routes are going to LPM:

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

Finaly, on scale line card, regardless the profile enabled, the /32 are stored in LEM and not eTCAM:

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
With both host-optimized and internet-optimized profile on base cards, we will see these prefixes moved to the LEM.

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

On scale line cards, the only the /32s are going to LEM, the rest (including the 500,000 /24s) will be pushed to the external TCAM:

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
[...]

Current Usage
    NPU-0
        Total In-Use                : 127      (0 %)
        iproute                     : <mark>24</mark>       (0 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 102      (0 %)
[...]

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources lpm location 0/6/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 118638
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
[...]

Current Usage
    NPU-0
        Total In-Use                : 144      (0 %)
        iproute                     : <mark>0</mark>        (0 %)
        ip6route                    : 117      (0 %)
        ipmcroute                   : 50       (0 %)
[...]

RP/0/RP0/CPU0:NCS5500-614#sh contr npu resources exttcamipv4 location 0/6/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
[...]

Current Usage
    NPU-0
        Total In-Use                : 500010   (24 %)
        iproute                     : <mark>500010</mark>   (24 %)
        ipmcroute                   : 0        (0 %)
[...]

RP/0/RP0/CPU0:NCS5500-614#
</code>
</pre>
</div>

__300k IPv4 /23 routes__

In this third example, we announce 300,000 IPv4/23 prefixes.
With the Host-optimized profiles on base line cards, they will be moved to the LPM.

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

Only 261k /23 prefixes are creating a saturation (we removed the error messages reporting that extra entries have not been programmed in hardware because the LPM capacity is exceeded).

Let's enable the Internet-optimized profiles (and reload).
This time, the 300,000 IPv4/23 will be split in two, making 600,000 IPv4/24 that will be moved to the LEM:

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

The same example with scale line cards will not be dependant on the optimized profile activated, all the IPv4/23 routes will be stored in the external TCAM:

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
You got it, so let's skip the host-optimized profile on base cards where these 100k will be stored in the LPM and let's skip also the scale cards where these routes will be pushed to the external TCAM, and let's focus on the behavior with base cards running an internet-optimized profile.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
aaaaaaa
<mark>bbb</mark>
</code>
</pre>
</div>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
aaaaaaa
<mark>bbb</mark>
</code>
</pre>
</div>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
aaaaaaa
<mark>bbb</mark>
</code>
</pre>
</div>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
aaaaaaa
<mark>bbb</mark>
</code>
</pre>
</div>

<div class="highlighter-rouge">
<pre class="highlight">
<code>
aaaaaaa
<mark>bbb</mark>
</code>
</pre>
</div>

### Estimation with Internet Distribution

### Production verification


