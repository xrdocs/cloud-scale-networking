---
published: false
date: '2018-07-12 16:34 +0200'
title: Security ACL on NCS5500 (Part1)
author: Nicolas Fevrier
excerpt: First part of a series of posts on NCS5500 Access-lists
Top: null
position: top
---
{% include toc icon="table" title="NCS5500 Security Access-lists - Part1" %} 

## Introduction

First article on the Security Access-List implementation on the NCS5500 series. We will dedicate a separate post on the Hybrid-ACL (also known as Scale-ACL or Object-Based-ACL). 

While hybrid-ACL will be only supported on -SE systems with external TCAM, the traditional security ACL can be used on all systems and line cards of the portfolio. They are available in ingress, egress, for IPv4, IPv6 and L2.

Please note: we don’t cover access-list used for route-filtering in this document. We don’t cover Access-list Based Forwarding feature, nor SPAN (packet capture / replication) based on ACL either. We only intend to cover security ACL aimed at filtering packets going through or to the routing device.


## Basic notions on ACLs

An access-list is applied under on interface statement. It contains an address-family, an ACL identifier (or name) and a direction.

![acl-format.png]({{site.baseurl}}/images/acl-format.png)

An access-list is made of access-list entries (ACEs). The scale, both in term of ACL and ACE, will depend on the type of interface, the address-family and the direction.

The first part of the ACL defines the protocol-type L2, v4 or v6, and describes the name used to call it under the inferfaces. The following lines are representing the Access-list Entries (ACEs).

![ACE-ACL2.png]({{site.baseurl}}/images/ACE-ACL2.png)

If you don't use numbers to identify the lines when you configure your ACEs, the system will automatically assign numbers. They are multiple of 10 and increment line after line. After the creation, the operateur will be able to edit the ACL content, inserting entries with intermediate values or deleting entries with the appropriate value.

In ASR9000 or CRS, it was possible to re-sequence the ACEs but it's not supported with NCS5500.


## Interface/ACL Support (status in IOS XR 6.2.3 / 6.5.1)

Where can be "used" these ACLs ? We support L2 and L3 ACL but “conditions may apply"

- Ingress IPv4 ACLs are supported on L3 physical, bundles, sub-interfaces and bundled sub-interfaces, but also on BVI interfaces
- Ingress IPv6 ACLs are supported on L3 physical, bundles, sub-interfaces and bundled sub-interfaces, but also on BVI interfaces
- Egress IPv4 ACLs are supported on L3 physical and bundle interfaces but also on BVI interfaces
- Egress IPv6 ACLs are supported on L3 physical and bundle interfaces but also on BVI interfaces
- Egress IPv4 or IPv6 ACLs are NOT supported on L3 sub-interfaces or bundled sub-interfaces (but if you apply the ACL on the physical or bundle, all packets on the sub-interfaces will be handled by this ACL)
- It’s no possible to apply an L2 ACL on an IPv4/IPv6 (L3) interface or vice versa
- Ingress L2 ACLs are supported but not egress L2 ACLs
- Ranges are supported but only for source-port only

Let’s summarise:

| Interface Type | Direction | AF | Suppport ? |
|:-----:|:-----:|:-----:|:-----:|
|  L3 Physical | Ingress | IPv4 | YES |
|  L3 Physical | Ingress | IPv6 | YES |
|  L3 Physical | Egress | IPv4 | YES |
|  L3 Physical | Egress | IPv6 | YES |
|  L3 Physical | Ingress | L2 | NO |
|  L3 Physical | Egress | L2 | NO |
|  L3 Bundle | Ingress | IPv4 | YES |
|  L3 Bundle | Ingress | IPv6 | YES |
|  L3 Bundle | Egress | IPv4 | YES |
|  L3 Bundle | Egress | IPv6 | YES |
|  L3 Bundle | Ingress | L2 | NO |
|  L3 Bundle | Egress | L2 | NO |
|  L3 Sub-interface | Ingress | IPv4 | YES |
|  L3 Sub-interface | Ingress | IPv6 | YES |
|  L3 Sub-interface | Egress | IPv4 | NO |
|  L3 Sub-interface | Egress | IPv6 | NO |
|  L3 Bundled Sub-interface | Ingress | IPv4 | YES |
|  L3 Bundled Sub-interface | Ingress | IPv6 | YES |
|  L3 Bundled Sub-interface | Egress | IPv4 | NO |
|  L3 Bundled Sub-interface | Egress | IPv6 | NO |
|  L3 Bundled Sub-interface | Ingress | L2 | NO |
|  L3 Bundle Sub-interface | Egress | L2 | NO |
|  BVI | Ingress | IPv4 | YES |
|  BVI | Ingress | IPv6 | YES |
|  BVI | Egress | IPv4 | YES |
|  BVI | Egress | IPv6 | NO |
|  Tunnel | Ingress | IPv4 | Partial |
|  Tunnel | Ingress | IPv6 | NO |
|  Tunnel | Egress | IPv4 | Partial |
|  Tunnel | Egress | IPv6 | NO |
|  L2 | Ingress | IPv4 | NO |
|  L2 | Ingress | IPv6 | NO |
|  L2 | Egress | IPv4 | NO |
|  L2 | Egress | IPv6 | NO |
|  L2 | Ingress | L2 | YES |
|  L2 | Egress | L2 | NO |


## Scale

The number of ACLs and ACEs we support is expressed per NPU (Qumran-MX, Jericho, Jericho+). Since the ACLs are applied on ports, we invite you to check the former blog post describing the port to NPU assignments.

Also, keep in mind that an ACL applied to a bundle interface where the port members span over multiple NPU will see the ACL/ACEs replicated on all the participating NPUs.

By default (that mean without changing the hardware profiles), we support simultaneously up to:
- max 31 unique attached ingress ACLs per NPU
- max 255 unique attached egress ACLs per NPU
- max 4000 attached ingress IPv4 ACEs per LC
- max 4000 attached egress IPv4 ACEs per LC
- max 2000 attached ingress IPv6 ACEs per LC
- max 2000 attached egress IPv6 ACEs per LC
- max 2000 attached ingress L2 ACEs per LC

Note that it's possible to configure much more, the limits listed above are for the ACL/ACE attached to interfaces.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#show access-lists ipv4 maximum detail

Default max configurable acls :16000
Default max configurable aces :350000
Current configured acls       :22
Current configured aces       :93455
Current max configurable acls :16000
Current max configurable aces :350000
Max configurable acls         :16000
Max configurable aces         :350000
RP/0/RP0/CPU0:5500-6.3.2#show access-lists ipv6 maximum detail

Default max configurable acls :16000
Default max configurable aces :350000
Current configured acls       :1
Current configured aces       :1003
Current max configurable acls :16000
Current max configurable aces :350000
Max configurable acls         :16000
Max configurable aces         :350000
RP/0/RP0/CPU0:5500-6.3.2#
</code>
</pre>
</div>


## Match support, Parameters and Edition

### Edition

When using traditional / "flat" ACL, it's possible to edit the ACEs in-line. But when using object-groups (defined in a following section), it's an atomic process where the new ACE replaces the existing one.

### Range

We support range statement

ipv4 access-list test-range-24
10 permit tcp host 1.2.3.4 range 130 150 host 2.3.4.5
20 permit tcp host 1.2.3.4 range 230 250 host 2.3.4.5
30 permit tcp host 1.2.3.4 range 330 350 host 2.3.4.5
40 permit tcp host 1.2.3.4 range 430 450 host 2.3.4.5
50 permit tcp host 1.2.3.4 range 530 550 host 2.3.4.5
60 permit tcp host 1.2.3.4 range 630 750 host 2.3.4.5
70 permit tcp host 1.2.3.4 range 730 750 host 2.3.4.5
80 permit tcp host 1.2.3.4 range 830 850 host 2.3.4.5
90 permit tcp host 1.2.3.4 range 930 950 host 2.3.4.5
100 permit tcp host 1.2.3.4 range 1030 1050 host 2.3.4.5
110 permit tcp host 1.2.3.4 range 1130 1150 host 2.3.4.5
120 permit tcp host 1.2.3.4 range 1230 1250 host 2.3.4.5
130 permit tcp host 1.2.3.4 range 1330 1350 host 2.3.4.5
140 permit tcp host 1.2.3.4 range 1430 1450 host 2.3.4.5
150 permit tcp host 1.2.3.4 range 1530 1550 host 2.3.4.5
160 permit tcp host 1.2.3.4 range 1630 1750 host 2.3.4.5
170 permit tcp host 1.2.3.4 range 1730 1750 host 2.3.4.5
180 permit tcp host 1.2.3.4 range 1830 1850 host 2.3.4.5
190 permit tcp host 1.2.3.4 range 1930 1950 host 2.3.4.5
200 permit tcp host 1.2.3.4 range 2030 2050 host 2.3.4.5
210 permit tcp host 1.2.3.4 range 2130 2150 host 2.3.4.5
220 permit tcp host 1.2.3.4 range 2230 2250 host 2.3.4.5
230 permit tcp host 1.2.3.4 range 2330 2350 host 2.3.4.5
240 permit tcp host 1.2.3.4 range 2430 2450 host 2.3.4.5
250 permit tcp host 1.2.3.4 range 2530 2550 host 2.3.4.5
260 permit tcp host 1.2.3.4 range 2630 2650 host 2.3.4.5
270 permit tcp host 1.2.3.4 range 2730 2750 host 2.3.4.5
280 permit tcp host 1.2.3.4 range 2830 2850 host 2.3.4.5
290 permit tcp host 1.2.3.4 range 2930 2950 host 2.3.4.5
300 permit tcp host 1.2.3.4 range 3030 3050 host 2.3.4.5


### match statements

The following protocols can be matched:
- IGMP
	- type
- ICMP
	- type
    - code
- UDP
	- protocol name or port number
    - DSCP / precedence
    - fragments
    - log
    - icmp-off
    - packet length (eq or range)
    - ttl
- TCP
	- protocol name or port number
    - DSCP / precedence
    - established
    - icmp-off
    - packet length (eq or range)
    - ttl
    
Check [https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html#reference_7C68561395FF4CE1902EF920B47FA254](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html#reference_7C68561395FF4CE1902EF920B47FA254) for a complete list.

### enable-set-ttl
https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html#id_60681

### ttl match

<div class="highlighter-rouge">
<pre class="highlight">
<code>
hw-module profile tcam format access-list ipv4 src-addr src-port enable-set-ttl ttl-match
hw-module profile tcam format access-list ipv4 dst-addr dst-port enable-set-ttl ttl-match 
</code>
</pre>
</div>

### frag

### log





## Memory space

Traditional / non-hybrid ACLs are stored in the internal TCAM, even on -SE systems.

If you check memory utilisation in 6.1.x:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-1-614#sh contrnpuinternaltcamloc0/7/CPU0 | i "(size|NPU|==|Id)"
NPU 0:
==================================================================================
BankId       Key EntrySize      Free          InUse         Nof DBs       Owner
        DB Id         DB InUse      Prefix
==================================================================================
0             size_160_bits       2043          5             8             pmf-0
1             size_160_bits       2047          1             1             pmf-1
2\3           size_320_bits       1972          76            3             pmf-0
4\5           size_320_bits       2020          28            1             pmf-0
12            size_160_bits       126           2             1             pmf-1
13            size_160_bits       115           13            1             pmf-0
14            size_160_bits       118           10            1             egress_acl
</code>
</pre>
</div>

If you do it on 6.3 or later:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-2-622#sh contrnpuinternaltcamlocation0/7/CPU0
InternalTCAM ResourceInformation
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0\1    320b   pmf-0       2006     36      7    INGRESS_LPTS_IPV4
0    0\1    320b   pmf-0       2006     2       12   INGRESS_RX_ISIS
0    0\1    320b   pmf-0       2006     2       32   INGRESS_QOS_IPV6
0    0\1    320b   pmf-0       2006     2       34   INGRESS_QOS_L2
0    2      160b   pmf-0       2044     2       31   INGRESS_QOS_IPV4
0    2      160b   pmf-0       2044     1       33   INGRESS_QOS_MPLS
0    2      160b   pmf-0       2044     1       42   INGRESS_ACL_L2
0    3      160b   egress_acl  2022     10      3    EGRESS_RECEIVE
0    3      160b   egress_acl  2022     16      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2024     24      8    INGRESS_LPTS_IPV6
0    6      160b   Free        2048     0       0
0    7      160b   Free        2048     0       0
0    8      160b   Free        2048     0       0
0    9      160b   Free        2048     0       0
0    10     160b   Free        2048     0       0
0    11     160b   Free        2048     0       0
0    12     160b   pmf-1       90       37      11   INGRESS_RX_L2
0    12     160b   pmf-1       90       1       13   INGRESS_MCAST_IPV4_ASM
0    13     160b   pmf-0       112      2       10   INGRESS_DHCP
0    13     160b   pmf-0       112      13      26   INGRESS_MPLS
0    13     160b   pmf-0       112      1       41   INGRESS_EVPN_AA_ESI_TO_FBN_DB
0    14     160b   Free        128      0       0
0    15     160b   Free        128      0       0
</code>
</pre>
</div>

Now with 1000 ACEs.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-2-622#sh contrnpuinternaltcamlocation0/7/CPU0
InternalTCAM ResourceInformation
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0\1    320b   pmf-0       2005     37      7    INGRESS_LPTS_IPV4
0    0\1    320b   pmf-0       2005     2       12   INGRESS_RX_ISIS
0    0\1    320b   pmf-0       2005     2       32   INGRESS_QOS_IPV6
0    0\1    320b   pmf-0       2005     2       34   INGRESS_QOS_L2
0    2      160b   pmf-0       2044     2       31   INGRESS_QOS_IPV4
0    2      160b   pmf-0       2044     1       33   INGRESS_QOS_MPLS
0    2      160b   pmf-0       2044     1       42   INGRESS_ACL_L2
0    3      160b   egress_acl  2022     10      3    EGRESS_RECEIVE
0    3      160b   egress_acl  2022     16      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2024     24      8    INGRESS_LPTS_IPV6
0    6      160b   pmf-0       997      1051    16   INGRESS_ACL_L3_IPV4
0    7      160b   Free        2048     0       0
0    8      160b   Free        2048     0       0
0    9      160b   Free        2048     0       0
0    10     160b   Free        2048     0       0
0    11     160b   Free        2048     0       0
0    12     160b   pmf-1       90       37      11   INGRESS_RX_L2
0    12     160b   pmf-1       90       1       13   INGRESS_MCAST_IPV4_ASM
0    13     160b   pmf-0       112      2       10   INGRESS_DHCP
0    13     160b   pmf-0       112      13      26   INGRESS_MPLS
0    13     160b   pmf-0       112      1       41   INGRESS_EVPN_AA_ESI_TO_FBN_DB
0    14     160b   Free        128      0       0
0    15     160b   Free        128      0       0
</code>
</pre>
</div>

## Sharing / Unique

By default again, it's possible to share access-lists in ingress but not in egress.

What does it mean exactly? Let's take 2 interfaces handled by the same NPU: Hu0/7/0/1 and Hu0/7/0/2.

Before applying the ACLs:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5500-6.3.2#sh contr npu internaltcam loc 0/7/CPU0

Internal TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0\1    320b   pmf-0       1987     49      7    INGRESS_LPTS_IPV4
0    0\1    320b   pmf-0       1987     8       12   INGRESS_RX_ISIS
0    0\1    320b   pmf-0       1987     2       32   INGRESS_QOS_IPV6
0    0\1    320b   pmf-0       1987     2       34   INGRESS_QOS_L2
0    2      160b   pmf-0       2044     2       31   INGRESS_QOS_IPV4
0    2      160b   pmf-0       2044     1       33   INGRESS_QOS_MPLS
0    2      160b   pmf-0       2044     1       42   INGRESS_ACL_L2
0    3      160b   egress_acl  2031     17      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2013     35      8    INGRESS_LPTS_IPV6
0    6      160b   <mark>Free</mark>        2048     0       0
0    7      160b   <mark>Free</mark>        2048     0       0
0    8      160b   <mark>Free</mark>        2048     0       0
0    9      160b   <mark>Free</mark>        2048     0       0
0    10     160b   <mark>Free</mark>        2048     0       0
0    11     160b   <mark>Free</mark>        2048     0       0
0    12     160b   pmf-1       30       41      11   INGRESS_RX_L2
0    12     160b   pmf-1       30       13      26   INGRESS_MPLS
0    12     160b   pmf-1       30       44      79   INGRESS_BFD_IPV4_NO_DESC_TCAM_T
0    13     160b   pmf-1       124      3       10   INGRESS_DHCP
0    13     160b   pmf-1       124      1       41   INGRESS_EVPN_AA_ESI_TO_FBN_DB
0    14     160b   Free        128      0       0
0    15     160b   Free        128      0       0
</code>
</pre>
</div>

In ingress, if we apply the same ACL of 1000 lines on these two interfaces.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5500-6.3.2#sh access-list ipv4 usage pfilter location 0/7/$

Interface : HundredGigE0/7/0/1
    Input  ACL : Common-ACL : N/A  ACL : test-1000
    Output ACL : N/A
Interface : HundredGigE0/7/0/2
    Input  ACL : Common-ACL : N/A  ACL : test-1000
    Output ACL : N/A
RP/0/RP0/CPU0:TME-5500-6.3.2#sh contr npu internaltcam loc 0/7/CPU0

Internal TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0\1    320b   pmf-0       1987     49      7    INGRESS_LPTS_IPV4
0    0\1    320b   pmf-0       1987     8       12   INGRESS_RX_ISIS
0    0\1    320b   pmf-0       1987     2       32   INGRESS_QOS_IPV6
0    0\1    320b   pmf-0       1987     2       34   INGRESS_QOS_L2
0    2      160b   pmf-0       2044     2       31   INGRESS_QOS_IPV4
0    2      160b   pmf-0       2044     1       33   INGRESS_QOS_MPLS
0    2      160b   pmf-0       2044     1       42   INGRESS_ACL_L2
0    3      160b   egress_acl  2031     17      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2013     35      8    INGRESS_LPTS_IPV6
0    <mark>6      160b   pmf-0       997      1051    16   INGRESS_ACL_L3_IPV4</mark>
0    7      160b   Free        2048     0       0
0    8      160b   Free        2048     0       0
0    9      160b   Free        2048     0       0
0    10     160b   Free        2048     0       0
0    11     160b   Free        2048     0       0
0    12     160b   pmf-1       30       41      11   INGRESS_RX_L2
0    12     160b   pmf-1       30       13      26   INGRESS_MPLS
0    12     160b   pmf-1       30       44      79   INGRESS_BFD_IPV4_NO_DESC_TCAM_T
0    13     160b   pmf-1       124      3       10   INGRESS_DHCP
0    13     160b   pmf-1       124      1       41   INGRESS_EVPN_AA_ESI_TO_FBN_DB
0    14     160b   Free        128      0       0
0    15     160b   Free        128      0       0
</code>
</pre>
</div>

We note that 1051 entries are consumed for these two access-lists.

We remove the ingress ACLs and apply the same on egress this time:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5500-6.3.2#sh access-list ipv4 usage pfilter location 0/7/CPU0

Interface : HundredGigE0/7/0/1
    Input ACL : N/A
    Output ACL : test-1000
Interface : HundredGigE0/7/0/2
    Input ACL : N/A
    Output ACL : test-1000
RP/0/RP0/CPU0:TME-5500-6.3.2#sh contr npu internaltcam loc 0/7/CPU0

Internal TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0\1    320b   pmf-0       1987     49      7    INGRESS_LPTS_IPV4
0    0\1    320b   pmf-0       1987     8       12   INGRESS_RX_ISIS
0    0\1    320b   pmf-0       1987     2       32   INGRESS_QOS_IPV6
0    0\1    320b   pmf-0       1987     2       34   INGRESS_QOS_L2
0    2      160b   pmf-0       2044     2       31   INGRESS_QOS_IPV4
0    2      160b   pmf-0       2044     1       33   INGRESS_QOS_MPLS
0    2      160b   pmf-0       2044     1       42   INGRESS_ACL_L2
0    <mark>3      160b   egress_acl  0        2031    1    EGRESS_ACL_IPV4</mark>
0    3      160b   egress_acl  0        17      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2013     35      8    INGRESS_LPTS_IPV6
0    6      160b   Free        2048     0       0
0    <mark>7      160b   egress_acl  1889     159     1    EGRESS_ACL_IPV4</mark>
0    8      160b   Free        2048     0       0
0    9      160b   Free        2048     0       0
0    10     160b   Free        2048     0       0
0    11     160b   Free        2048     0       0
0    12     160b   pmf-1       30       41      11   INGRESS_RX_L2
0    12     160b   pmf-1       30       13      26   INGRESS_MPLS
0    12     160b   pmf-1       30       44      79   INGRESS_BFD_IPV4_NO_DESC_TCAM_T
0    13     160b   pmf-1       124      3       10   INGRESS_DHCP
0    13     160b   pmf-1       124      1       41   INGRESS_EVPN_AA_ESI_TO_FBN_DB
0    14     160b   Free        128      0       0
0    15     160b   Free        128      0       0
</code>
</pre>
</div>

We can see that each ACL applied on egress is counted each time, so it spread between bank #3 and bank #7.

In summary, we are sharing the ACL on ingress but not on egress.

## Statistics

Counters

## Misc 

### No resequencing

It's possible to resequence ACL for prefix-list but not for security ACLs.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#resequence ?
  prefix-list  Prefix lists
RP/0/RP0/CPU0:5500-6.3.2#
</code>
</pre>
</div>

#### Copy is possible

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5500-6.3.2#copy access-list ?
  ethernet-service  Copy Ethernet Service access list
  ipv4              Copy IPv4 access list
  ipv6              Copy IPv6 access list
RP/0/RP0/CPU0:TME-5500-6.3.2#copy access-list ipv4 test-range-24 new-acl
RP/0/RP0/CPU0:TME-5500-6.3.2#
</code>
</pre>
</div>

### hw-module profile

Enable an IPv4 egress ACL on BVI
RP/0/RP0/CPU0:router(config)# hw-module profile acl egress layer3 interface-based

Enable permit statistics for the egress ACL (by default, only deny statistics are shown)
RP/0/RP0/CPU0:router(config)# hw-module profile stats acl-permit


## Resources

CCO guides: Implementing ACLs

[https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/63x/b-ip-addresses-configuration-guide-ncs5500-63x/b-ip-addresses-configuration-guide-ncs5500-63x_chapter_010.html](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/63x/b-ip-addresses-configuration-guide-ncs5500-63x/b-ip-addresses-configuration-guide-ncs5500-63x_chapter_010.html)

CCO guide: ACL commands

[https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html)

## Thanks :)

Thanks a lot to Puneet Kalra for his help on this post.
