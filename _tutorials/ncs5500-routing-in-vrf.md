---
published: true
date: '2018-04-03 01:25 +0200'
title: NCS5500 Routing in VRF
author: Nicolas Fevrier
excerpt: Is it possible to run the Internet Feed inside a VRF/VPN in an NCS5500 ?
tags:
  - ncs5500
  - iosxr
  - xr
  - internet
  - vrf
Position: hiddenn
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E08 NCS5500 Routes in VRF

### Previously on "Understanding NCS5500 Resources"

In previous posts... Well ok, you got it now. You can check all the former articles [in the page here](https://xrdocs.github.io/cloud-scale-networking/tutorials/). We presented the different platforms, based on Qumran-MX, Jericho and Jericho+. We detailed all the mechanisms used to optimized the routes sorting inside the various memories and we also detailed the impact of features like URPF.

Last week, we've been asked if it's possible "to run the Internet Feed inside a VRF/VPN".

It's indeed a very good question since we used to have some platforms where the scale of routes inside VRF was significantly different than the capability in Global Routing Table.

Short answer: yes we support it. But it's important to set it up correctly to avoid surprises.

It has been explain extensively in the former posts, so we suppose you are now familiar with the logic of sorting and storing routes in different memories depending on the product (whether or not we have external TCAM) and on the prefix length.

### Let's configure it

Let's configure an interface and advertise 85k routes (IPv4/27). For this example, we will use a Jericho line cards with eTCAM (NC55-24X100G-SE) running IOS XR 6.3.2.

Note: L3VPN was available in some specific images but is officially supported only in 6.3.2. Before this release, it was possible to configure VRF-lite. What will be described below applies for both.
{: .notice--info}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
vrf TEST
address-family ipv4 unicast
!
!
interface HundredGigE0/7/0/2
cdp
vrf TEST
ipv4 address 192.168.21.1 255.255.255.0
!
router bgp 100
vrf TEST
  rd 113579:13579
  address-family ipv4 unicast
  !
  neighbor 192.168.21.2
   remote-as 100
   update-source HundredGigE0/7/0/2
   address-family ipv4 unicast
    route-policy ROUTE-FILTER in
    maximum-prefix 8000000 75
    route-policy PERMIT-ANY out
   !
  !
!
!
</code>
</pre>
</div>

And the BGP routes received from my neighbor:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-1-6.3.2#sh bgp vrf TEST summary

BGP VRF TEST, state: Active
BGP Route Distinguisher: 113579:13579
VRF ID: 0x60000003
BGP router identifier 1.1.1.1, local AS number 100
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000003   RD version: 85622
BGP main routing table version 85622
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 85622/0

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker           85622      85622      85622      85622       85622       85622

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.21.2      0   100  355498      44    85622    0    0 00:39:18      <mark>85614</mark>

RP/0/RP0/CPU0:5508-1-6.3.2#
</code>
</pre>
</div>

These routes being IPv4/27, they will be stored in external TCAM.

Let's examine the memory resources:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh contr npu resources all loc 0/7/CPU0

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
        Total In-Use                : 85645    (11 %)
        iproute                     : 40       (0 %)
        ip6route                    : 0        (0 %)
        <mark>mplslabel</mark>                   : <mark>85614</mark>    <mark>(11 %)</mark>

-- SNIP --

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 251311
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 52       (0 %)
        iproute                     : 0        (0 %)
        ip6route                    : 38       (0 %)
        ipmcroute                   : 1        (0 %)

-- SNIP --

HW Resource Information
    Name                            : encap

OOR Information
    NPU-0
        Estimated Max Entries       : 80000
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 4        (0 %)
        ipnh                        : 2        (0 %)
        ip6nh                       : 2        (0 %)
        mplsnh                      : 0        (0 %)

-- SNIP --

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 85623    (4 %)
        <mark>iproute</mark>                     : <mark>85635</mark>    <mark>(4 %)</mark>

-- SNIP --

HW Resource Information
    Name                            : fec

OOR Information
    NPU-0
        Estimated Max Entries       : 126976
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 65       (0 %)
        ipnhgroup                   : 55       (0 %)
        ip6nhgroup                  : 10       (0 %)

-- SNIP --

HW Resource Information
    Name                            : ecmp_fec

OOR Information
    NPU-0
        Estimated Max Entries       : 4096
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 0        (0 %)
        ipnhgroup                   : 0        (0 %)
        ip6nhgroup                  : 0        (0 %)
 -- SNIP --
</code>
</pre>
</div>

As expected we found 85635 iproute in the externaltcamv4, but also we notice the presence of 85614 mplslabel entries in the LEM memory. So for each prefix learnt in the VRF TEST, we associate by default a label and it consumes one entry in the LEM even if it's not a IPv4/32.

This is indeed the default behavior: per-prefix label allocation...

If your design permits it (and it should be the case in 99% of the times), we advise you modify the label allocation mode to "per-vrf".

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-1-6.3.2#conf
RP/0/RP0/CPU0:5508-1-6.3.2(config)#
RP/0/RP0/CPU0:5508-1-6.3.2(config)#router bgp 100
RP/0/RP0/CPU0:5508-1-6.3.2(config-bgp)# vrf TEST
RP/0/RP0/CPU0:5508-1-6.3.2(config-bgp-vrf)# address-family ipv4 unicast
RP/0/RP0/CPU0:5508-1-6.3.2(config-bgp-vrf-af)#  label mode per-vrf
RP/0/RP0/CPU0:5508-1-6.3.2(config-bgp-vrf-af)#commit
RP/0/RP0/CPU0:5508-1-6.3.2(config-bgp-vrf-af)#end
RP/0/RP0/CPU0:5508-1-6.3.2#
</code>
</pre>
</div>

Let's now check the impact on LEM:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-1-6.3.2#sh contr npu resources all loc 0/7/CPU0

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
        Total In-Use                : 46       (0 %)
        iproute                     : 40       (0 %)
        ip6route                    : 0        (0 %)
        <mark>mplslabel</mark>                   : <mark>1</mark>        <mark>(0 %)</mark>

-- SNIP --

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 85623    (4 %)
        iproute                     : 85635    (4 %)

-- SNIP --       
</code>
</pre>
</div>

It changes everthing now that we allocate only one entry for the MPLS label. For instance, it permits to push easily a full internet view and much more (for instance 435k extra host routes).

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-1-6.3.2#sh bgp vrf TEST summary

BGP VRF TEST, state: Active
BGP Route Distinguisher: 113579:13579
VRF ID: 0x60000003
BGP router identifier 1.1.1.1, local AS number 100
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000003   RD version: 1877292
BGP main routing table version 1877292
BGP NSR Initial initsync version 1 (Reached)
BGP NSR/ISSU Sync-Group versions 1877292/0

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker         1877292    1877292    1877292    1877292     1877292     1877292

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.21.2      0   100 1261510     148  1877292    0    0 02:18:40    1186410

RP/0/RP0/CPU0:5508-1-6.3.2#sh route vrf TEST sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            1          0          0           240
connected                        1          0          0           240
dagr                             0          0          0           0
bgp 100                          1186410    0          0           284738400
static                           1          0          0           240
Total                            1186413    0          0           284739120

RP/0/RP0/CPU0:5508-1-6.3.2#sh contr npu resources all loc 0/7/CPU0

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
        Total In-Use                : 434784   (55 %)
        iproute                     : 434793   (55 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 1        (0 %)

-- SNIP --

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 251311
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 52       (0 %)
        iproute                     : 0        (0 %)
        ip6route                    : 38       (0 %)
        ipmcroute                   : 1        (0 %)

-- SNIP --

HW Resource Information
    Name                            : encap

OOR Information
    NPU-0
        Estimated Max Entries       : 80000
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 4        (0 %)
        ipnh                        : 2        (0 %)
        ip6nh                       : 2        (0 %)
        mplsnh                      : 0        (0 %)

-- SNIP --

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 2048000
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 751665   (37 %)
        iproute                     : 751677   (37 %)
-- SNIP --
</code>
</pre>
</div>

Also we can check how a non-eTCAM line cards (NC55-36X100G) can do with the full view (we just filter the IPv4/32 from the same BGP peer:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5508-1-6.3.2#sh run route-policy ROUTE-FILTER

route-policy ROUTE-FILTER
  if destination in (0.0.0.0/0 le 31) then
    pass
  else
    drop
  endif
end-policy

RP/0/RP0/CPU0:5508-1-6.3.2#
RP/0/RP0/CPU0:5508-1-6.3.2#sh route vrf TEST summary

Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            1          0          0           240
connected                        1          0          0           240
dagr                             0          0          0           0
bgp 100                          751657     0          0           180397680
static                           1          0          0           240
Total                            <mark>751660</mark>     0          0           180398400

RP/0/RP0/CPU0:5508-1-6.3.2#
RP/0/RP0/CPU0:5508-1-6.3.2#sh dpa resources iproute loc 0/2/CPU0

"iproute" DPA Table (Id: 24, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution
Prefix   Actual       Prefix   Actual
 /0       4            /1       0
 /2       0            /3       0
 /4       4            /5       0
 /6       0            /7       0
 /8       15           /9       13
 /10      35           /11      106
 /12      285          /13      550
 /14      1066         /15      1880
 /16      13419        /17      7773
 /18      13636        /19      25026
 /20      38261        /21      43073
 /22      80751        /23      67073
 /24      376990       /25      567
 /26      2032         /27      4863
 /28      15599        /29      16868
 /30      41736        /31      52
 /32      39
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3           NPU-4           NPU-5
                          In Use: 751716          751716          751716          751716          751716          751716
                          
RP/0/RP0/CPU0:5508-1-6.3.2#sh contr npu resources all loc 0/2/CPU0

HW Resource Information
    Name                            : lem

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
        OOR State Change Time       : 2018.Apr.02 10:32:37 PDT
        
-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 377026   (48 %)
        iproute                     : 349729   (44 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 1        (0 %)
        
-- SNIP --

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 418492
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2018.Apr.02 09:39:01 PDT
        
-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : 374731   (90 %)
        iproute                     : 374687   (90 %)
        ip6route                    : 38       (0 %)
        ipmcroute                   : 1        (0 %)
        
-- SNIP --
</code>
</pre>
</div>

We notice inconsistencies in the LEM numbers between Total-in-use and the total below. It's a cosmetic issue that will be handled in the next release.
{: .notice--info}

So, we can verify with this output that we are not consuming entries with mplslabel in LEM for each prefix.


### Conclusion

It's possible to learn a large number of routes in VRF but it's important to change the default label allocation mode to per-vrf, otherwise we will create one label entry for each prefix learnt.

Thanks to Lukas Mergenthaler
