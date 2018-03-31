---
published: true
date: '2018-03-31 01:44 +0200'
title: 'NCS5500 URPF: Configuration and Impact on Scale'
author: Nicolas Fevrier
excerpt: >-
  In this article, we analyse the configuration and impact of URPF on NCS5500
  systems
position: top
tags:
  - ncs5500
  - urpf
  - iosxr
  - internet
  - scale
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E07 NCS5500 URPF Configuration and Impact on Scale

### Previously on "Understanding NCS5500 Resources"

In previous posts, we presented:
- the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/)  
- we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/)
- we covered [how IPv6 prefixes are stored in the same databases](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-07-understanding-ncs5500-resources-s01e03/).
- [we demonstrated in a video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-12-30-full-internet-view-on-base-ncs-5500-systems-s01e04/) how we can handle a full IPv4 and IPv6 Internet view on "Base" systems and line cards (i.e. without external TCAM, only using the LEM and LPM internal to the forwarding ASIC)
- in the fifth post, [we continued with a new video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2018-01-25-s01e05-large-routing-tables-on-scale-ncs-5500-systems/) where we demonstrated the very high scale we can reach on Jericho-based systems when using an external TCAM
- last post, [we introduced systems based on the Jericho+ forwarding ASIC](https://xrdocs.github.io/cloud-scale-networking/tutorials/Understanding-ncs5500-jericho-plus-systems/) and we detailed the routing distribution between the different memories and the scale they can achieve.

In this new episode, we will cover the impact of activating URPF on the NCS5500 routers.

### Definition

URPF stands for **Unicast Reverse Path Forwarding**.

Defintion from the CCO website: 

"_This security feature works by enabling a router to verify the reachability of the source address in packets being forwarded. This capability can limit the appearance of spoofed addresses on a network. If the source IP address is not valid, the packet is discarded._

_Unicast RPF in strict mode, the packet must be received on the interface that the router would use to forward the return packet._

_Unicast RPF in loose mode, the source address must appear in the routing table._"

It's a feature configured at the interface level.

### URPF relevancy

Regardless of the Forwarding ASIC (Qumran-MX, Jericho or Jericho+), the NCS5500 only supports URPF in loose mode.

Configuring URPF comes at a cost in term of scale on some of the NCS5500 family members. It will be detailed extensively in this article. That's why it's important to understand what are the benefits of enabling this feature.

As explained in the definition section above, the loose mode simply verify that source addresses of the packets received are in of the routable space. To bypass this "protection", it's fairly easy for an attacker to pick source addresses inside existing routes when forging the packet instead of totally random addresses.

We invite the operators to check how much traffic is currently dropped by the URPF loose mode if they have it enabled on production routers. 

Example: to check this on an ASR9000:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:Router#show cef drops | i RPF drops

  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :               0
  RPF drops            packets :           50065
  RPF drops            packets :               0
  RPF drops            packets :            1262
  RPF drops            packets :         3627918
  RPF drops            packets :            1262
  
-- SNIP --
</code>
</pre>
</div>

And compare these figures to the packet count per interface to understand how much traffic it represents. The impact it could have on route scale and the protection efficiency it offers need to be put in perspective before deciding if it is worth enabling URPF.

Now said, some other very good reasons to enable URPF loose mode exist. For example, it's a mandatory brick of a [Source-based Remotely Triggered Black Hole](https://www.cisco.com/c/dam/en_us/about/security/intelligence/blackhole.pdf) architecture (S-RTBH).

### NCS5500 Implementation

We don't support URPF strict mode and we have no plans to add it in the future on NCS5500. URPF loose mode is available on NCS5500 since IOS XR 6.2.2 for IPv4 and IPv6. The feature is supported on Jericho and Jericho+ systems, with or without eTCAM.

The configuration implies the deactivation of some profiles, different on "base" and "scale" systems. After this preliminary operation, the configuration is applied at the interface level.

Deactivating URPF on an interface implies to do it for both IPv4 and IPv6.

Allow-self-ping is the default mode and allow-default is not supported.

### Configuration and impact on base systems (no eTCAM)

On "base" systems (without external TCAM):

Since URPF requires two accesses to the LEM (lookup for source address then for destination address in the packet header), we have to disable the optimizations present by default or after a configuration:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
hw-module fib ipv4 scale host-optimized-disable
hw-module fib ipv6 scale internet-optimized-disable
</code>
</pre>
</div>

Note: depending on the IOS XR version, the options could be different and actually could be the opposite of "disable", be attentive at what is availabe in the CLI.
{: .notice--info}

With the optimization disabled and after the line cards / system reload, we have now:

![disable-optimizations.jpg]({{site.baseurl}}/images/disable-optimizations.jpg)

Important: with such mode, it will no longer be possible to handle a full internet view (v4+v6 or v4-only).
{: .notice--info}

The configuration can now be applied on the interfaces:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
hw-module fib ipv4 scale host-optimized-disable
hw-module fib ipv6 scale internet-optimized-disable
!
interface HundredGigE0/7/0/0
 ipv4 address 192.168.1.1 255.255.255.252
 ipv4 verify unicast source reachable-via any
 ipv6 verify unicast source reachable-via any
 ipv6 address 2001:10:1::1/64
</code>
</pre>
</div>

### Configuration and impact on scale Jericho systems (with eTCAM)

Now, let's consider the scale systems and line cards with Jericho ASICs:

The eTCAM is a 80bit memory and in normal condition we use it in two blocks of 40 bits to double the capacity. The first access being performed on the first half and the second access in the pipeline being done on the second half.

![double-cap.jpg]({{site.baseurl}}/images/double-cap.jpg)

With URPF, we need these two accesses to check source and destination, it's no longer possible to use the double capacity mode: it needs to be disabled.

![double-cap-disabled.jpg]({{site.baseurl}}/images/double-cap-disabled.jpg)

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-632(config)#hw-module tcam fib ipv4 scaledisable
RP/0/RP0/CPU0:NCS5508-632(config)#commit
</code>
</pre>
</div>

The impact on scale is significative since we lost 1M out of the 2M of the eTCAM capacity.

![urpf-impact.jpg]({{site.baseurl}}/images/urpf-impact.jpg)

Let's check with a large routing table (internet v4 + internet v6 + 435k host routes) what is the impact:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-6.3.2#sh bgp sum

BGP router identifier 1.1.1.1, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 1720749
BGP main routing table version 1720749
BGP NSR Initial initsync version 634456 (Reached)
BGP NSR/ISSU Sync-Group versions 1720749/0
BGP scan interval 60 secs
 
BGP is operating in STANDALONE mode.
 
 
Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker         1720749    1720749    1720749    1720749     1720749     1720749
 
Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.100.151   0  1000  706753  307270  1720749    0    0     5w0d     655487
192.168.100.152   0 45896  707144  307270  1720749    0    0     5w0d     656126
192.168.100.153   0  7018  705342  307270  1720749    0    0     5w0d     654330
192.168.100.154   0  1836  709963  307270  1720749    0    0     5w0d     658948
192.168.100.155   0 50300  687217  307270  1720749    0    0     5w0d     636208
192.168.100.156   0 50304  708316  307270  1720749    0    0     5w0d     657301
192.168.100.157   0 57381  708322  307270  1720749    0    0     5w0d     657307
192.168.100.158   0  4608  728503  812358  1720749    0    0     5w0d     677487
192.168.100.159   0  4777  717228  307270  1720749    0    0     5w0d     666213
192.168.100.160   0 37989  339686  307270  1720749    0    0     5w0d     288706
192.168.100.161   0  3549  705390  307270  1720749    0    0     5w0d     654376
192.168.100.163   0  8757  683499  307270  1720749    0    0     5w0d     632483
192.168.100.164   0  3257  705671  307270  1720749    0    0     5w0d     654661
192.168.100.166   0 10051 1186443  217145  1720749    0    0 00:28:05    1186410
 
RP/0/RP0/CPU0:5508-6.3.2#sh dpa resource iproute loc 0/7/CPU0
 
"iproute" DPA Table (Id: 24, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution
Prefix   Actual            Capacity    Prefix   Actual            Capacity
/0       3                 20           /1       0                 20
/2       0                 20           /3       0                 20
/4       3                 20           /5       0                 20
/6       0                 20           /7       0                 20
/8       16                20           /9       14                20
/10      37                204          /11      107               409
/12      288               818          /13      557               1636
/14      1071              3275         /15      1909              5732
/16      13572             42381        /17      8005              25387
/18      14055             42585        /19      25974             86603
/20      40443             127348       /21      45082             141679
/22      83722             231968       /23      71750             207173
/24      395142            1105590      /25      2085              4299
/26      3362              4504         /27      5736              3275
/28      15909             2866         /29      17377             6961
/30      42508             2866         /31      112               204
/32      435868            20
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: <mark>1224707</mark>         1224707         1224707         1224707
                 Create Requests
                           Total: 1224713         1224713         1224713         1224713
                         Success: 1224713         1224713         1224713         1224713
                 Delete Requests
                           Total: 6               6               6               6
                         Success: 6               6               6               6
                 Update Requests
                           Total: 341539          341539          341539          341539
                         Success: 341538          341538          341538          341538
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0
 
RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources lem loc 0/7/CPU0

HW Resource Information
    Name                            : lem
 
OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --
 
Current Usage
    NPU-0
        Total In-Use                : <mark>467163</mark>   <mark>(59 %)</mark>
        iproute                     : 435868   (55 %)
        ip6route                    : 31304    (4 %)
        mplslabel                   : 0        (0 %)

-- SNIP --
 
RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources lpm loc 0/7/CPU0

HW Resource Information
    Name                            : lpm
 
OOR Information
    NPU-0
        Estimated Max Entries       : 530552
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>5235</mark>     <mark>(1 %)</mark>
        iproute                     : 0        (0 %)
        ip6route                    : 5219     (1 %)
        ipmcroute                   : 0        (0 %)

-- SNIP --
 
RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources exttcamipv4 loc 0/7/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4
 
OOR Information
    NPU-0
        Estimated Max Entries       : <mark>2048000</mark>
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --
 
Current Usage
    NPU-0
        Total In-Use                : <mark>788839</mark>   <mark>(39 %)</mark>
        iproute                     : 788839   (39 %)
        ipmcroute                   : 0        (0 %)

-- SNIP --
 
RP/0/RP0/CPU0:5508-6.3.2#
</code>
</pre>
</div>

So we have 13 times Internet (coming from actual internet full views provided by different customers) and a lot of host routes (435k). In a Jericho + eTCAM card, before enabling URPF it occupies:
- LEM: **59%**
- LPM: **1%**
- eTCAM: **39%**

Let's now remove the double capacity mode and configure the URPF on interfaces

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-6.3.2#sh run | i hw-m

Building configuration...
RP/0/RP0/CPU0:5508-6.3.2#sh run int hu 0/7/0/0

interface HundredGigE0/7/0/0
cdp
ipv4 address 192.168.1.1 255.255.255.252
ipv6 address 2001:10:1::1/64
load-interval 30
flow ipv4 monitor fmm sampler fsm1 ingress
!

RP/0/RP0/CPU0:5508-6.3.2#conf

RP/0/RP0/CPU0:5508-6.3.2(config)#hw-module tcam fib ipv4 scaledisable

In order to activate this new scale, you must manually reload the chassis/all line cards
RP/0/RP0/CPU0:5508-6.3.2(config)#commit

RP/0/RP0/CPU0:5508-6.3.2(config)#end
RP/0/RP0/CPU0:5508-6.3.2#admin

root connected from 127.0.0.1 using console on 5500-6.3.2
sysadmin-vm:0_RP0# hw-module location 0/7 reload

Reload hardware module ? [no,yes] yes
result Card graceful reload request on 0/7 succeeded.
sysadmin-vm:0_RP0# exit

RP/0/RP0/CPU0:5508-6.3.2#conf
RP/0/RP0/CPU0:5508-6.3.2(config)#int hu 0/7/0/0
RP/0/RP0/CPU0:5508-6.3.2(config-if)# ipv4 verify unicast source reachable-via any
RP/0/RP0/CPU0:5508-6.3.2(config-if)# ipv6 verify unicast source reachable-via any
RP/0/RP0/CPU0:5508-6.3.2(config-if)#commit
RP/0/RP0/CPU0:5508-6.3.2(config-if)#end
RP/0/RP0/CPU0:5508-6.3.2#
RP/0/RP0/CPU0:5508-6.3.2#sh run | i hw-m

Building configuration...
hw-module tcam fib ipv4 scaledisable
RP/0/RP0/CPU0:5508-6.3.2#sh run int hu 0/7/0/0

interface HundredGigE0/7/0/0
cdp
ipv4 address 192.168.1.1 255.255.255.252
ipv4 verify unicast source reachable-via any
ipv6 verify unicast source reachable-via any
ipv6 address 2001:10:1::1/64
load-interval 30
flow ipv4 monitor fmm sampler fsm1 ingress
!
 
RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources lem loc 0/7/CPU0

HW Resource Information
    Name                            : lem
 
OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --
 
Current Usage
    NPU-0
        Total In-Use                : <mark>467173</mark>   <mark>(59 %)</mark>
        iproute                     : 435868   (55 %)
        ip6route                    : 31304    (4 %)
        mplslabel                   : 0        (0 %)

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources lpm loc 0/7/CPU0

HW Resource Information
    Name                            : lpm
 
OOR Information
    NPU-0
        Estimated Max Entries       : 171722
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>5234</mark>     <mark>(3 %)</mark>
        iproute                     : 0        (0 %)
        ip6route                    : 5218     (3 %)
        ipmcroute                   : 0        (0 %)

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources exttcamipv4 loc 0/7/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4
 
OOR Information
    NPU-0
        Estimated Max Entries       : <mark>1024000</mark>
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>788839</mark>   <mark>(77 %)</mark>
        iproute                     : 788839   (77 %)
        ipmcroute                   : 0        (0 %)

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#
</code>
</pre>
</div>

With URPF configured (and dual capacity mode disabled) and the same very large table we have:
- LEM: **59%**
- LPM: **3%**
- eTCAM: **77%**

In conclusion of this demo, enabling URPF implies the deactivation of the dual capacity mode and reduces by half the eTCAM memory. Nevertheless, routes are also stored in LEM and LPM. A very large internet table can still fit in the system, even if the room for growth is reduced.

### Configuration and impact on scale Jericho+ systems (with NG eTCAM)

The Jericho+ w/ eTCAM systems don't need to disable the dual capacity mode to enable URPF.

The same configuration than above can be re-used (except the hw-module commands).

The impact on scale is not null but is signficantly less than it was on the Jericho-based systems. Since the J+/eTCAM systems are qualified for 4M entries which is much less than its actual capacity, the 25% impact doesn't change the officially supported numbers: with URPF enabled we still support 4M routes in eTCAM.

### Verification

Packets dropped by URPF can be counted at the NPU level with:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-6.3.2#show contr npu stats traps-all instance 0 location 0/7/CPU0 | inc Rpf

<mark>RxTrapUcLooseRpfFail</mark>                          0    84   0x54        32035   0         0        
<mark>RxTrapUcStrictRpfFail</mark>                         0    137  0x89        32035   0         0   
</code>
</pre>
</div>

### Conclusion

URPF loose mode can be configured on all NCS5500 systems. On Jericho w/ eTCAM, the impact is significative but we demonstrated we still support a very large public table and a lot of host routes. On Jericho+ w/ eTCAM, URPF doesn't affect the supported scale of 4M entries.