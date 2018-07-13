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

Access-list are composed of deny or permit entries. If the entry denies an address or protocol, the NPU discards the packet and returns an Internet Control Message Protocol (ICMP) Host Unreachable message. It's possible to configure this behavior.


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

The number of ACLs and ACEs we support is expressed per NPU (Qumran-MX, Jericho, Jericho+). Since the ACLs are applied on ports, we invite you to check [the former blog post describing the port to NPU assignments](https://xrdocs.io/cloud-scale-networking/tutorials/2018-02-15-port-assignments-on-ncs5500-platforms/).

Also, keep in mind that an ACL applied to a bundle interface with port members spanning over multiple NPUs will see the ACL/ACEs replicated on all the participating NPUs.

By default (that mean without changing the hardware profiles), we support simultaneously up to:
- max 31 unique attached ingress ACLs per NPU
- max 255 unique attached egress ACLs per NPU
- max 4000 attached ingress IPv4 ACEs per LC
- max 4000 attached egress IPv4 ACEs per LC
- max 2000 attached ingress IPv6 ACEs per LC
- max 2000 attached egress IPv6 ACEs per LC
- max 2000 attached ingress L2 ACEs per LC

Note that it's possible to configure much more if they are not attached to interfaces.

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

When using traditional / "flat" ACLs, it's possible to edit the ACEs in-line. That means the ACL can be attached to interfaces, it's not needed to remove the ACL for the port to be able to edit it. But when using object-groups (defined in a following section), it's an atomic process where the new ACE replaces the existing one. It's logic considering that an object-based ACL is actually expanded in the iTCAM and the modification in a single line can be reflected in many lines changed in the hardware.

### Range

We support range statement but only with the limit of 23 range-IDs.

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

### ttl match

It's possible to match the TTL value in the IP header (both v4 and v6). We support exact values or ranges. For traditional (non-hybrid) ACL, it's not enabled by default and should be configured via a specific hardware profile

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 src-addr src-port enable-set-ttl ttl-match
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 dst-addr dst-port enable-set-ttl ttl-match 
</code>
</pre>
</div>

### enable-set-ttl

It's possible to match the TTL but also to manipulate this value. It's a bit outside of the scope of this post, so we invite you to check this URL if you are looking for more details.

[https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html#id_60681](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html#id_60681)

### frag match

We differentiate 3 types:
- non-fragmented
- initial fragments (with the port information)
- non-initial fragments (without the port number)
In the third category, we don't treat non-initial non-last and non-initial last fragments differently.

In NCS5500 platforms, we can match IPv4 fragments but we don't support IPv6 fragments.

Example:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
10 permit tcp 1.1.0.2/32 any dscp ef <mark>fragments</mark>
</code>
</pre>
</div>

If you define an ACL with L4 information (UDP or TCP ports for instance) and with "fragment" keyword, non-intital fragments can not be matched. It's expected since the packet no longer transport the port details, only an indication of fragment to help the re-assembly of the original packet at the destination host level.

The same ACL will be able to match initial fragments.

An L4 permit / deny without "fragment" keyword will be able to match non-fragmented and initial fragments while an L3 permit / deny without the keyword will be able to match all types of packets (non-fragmented, initial and non-initial fragments).

Some more details are available in the "Extended Access Lists with Fragment Control" section of this CCO document:

[https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/61x/b-ncs5500-ip-addresses-configuration-guide-61x/b-ncs5500-ip-addresses-configuration-guide-61x_chapter_01.html](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/61x/b-ncs5500-ip-addresses-configuration-guide-61x/b-ncs5500-ip-addresses-configuration-guide-61x_chapter_01.html)

### packet-length match

It's also possible to match on the packet length, which can be useful to tackle specific amplification attacks at the border of the internet (an alternative to using BGP Flowspec for example).



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

If you check the same on 6.3 or later:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500#sh contr npu internaltcam location 0/7/CPU0
</code>
</pre>
</div>

Check next sections for more CLI output.


## Sharing / Unique

### Shared ACLs

It's possible to share access-lists in ingress but not in egress.

What does it mean exactly? Let's take 2 interfaces handled by the same NPU: Hu0/7/0/1 and Hu0/7/0/2.

Before applying the ACLs:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#sh contr npu internaltcam loc 0/7/CPU0

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

All the banks from 6 to 10 are empty.

In ingress, if we apply the ACL "test-1000" (as the name imples, made of 1000 lines) on these two interfaces.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#sh access-list ipv4 usage pfilter location 0/7/$

Interface : HundredGigE0/7/0/1
    Input  ACL : Common-ACL : N/A  ACL : test-1000
    Output ACL : N/A
Interface : HundredGigE0/7/0/2
    Input  ACL : Common-ACL : N/A  ACL : test-1000
    Output ACL : N/A
RP/0/RP0/CPU0:5500-6.3.2#sh contr npu internaltcam loc 0/7/CPU0

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
0    6      160b   <mark>pmf-0</mark>       <mark>997</mark>      <mark>1051</mark>    16   <mark>INGRESS_ACL_L3_IPV4</mark>
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

We note that 1051 entries are consumed for these two access-lists. So the 1000 entries are just counted once even if the ACL is applied on multiple interfaces of the same NPU. That's what we qualified a "shared ACL".

Note: it's not showing exactly 1000 but 1051. The difference comes from internal entries automatically allocated by the system. They don't represent a significant number compared to the overall scale capability of the system.
{: .notice--info}

We remove the ingress ACLs and apply the same on egress this time:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#sh access-list ipv4 usage pfilter location 0/7/CPU0

Interface : HundredGigE0/7/0/1
    Input ACL : N/A
    Output ACL : test-1000
Interface : HundredGigE0/7/0/2
    Input ACL : N/A
    Output ACL : test-1000
RP/0/RP0/CPU0:5500-6.3.2#sh contr npu internaltcam loc 0/7/CPU0

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
0    3      160b   <mark>egress_acl</mark>  <mark>0</mark>        <mark>2031</mark>    1    <mark>EGRESS_ACL_IPV4</mark>
0    3      160b   egress_acl  0        17      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2013     35      8    INGRESS_LPTS_IPV6
0    6      160b   Free        2048     0       0
0    7      160b   <mark>egress_acl</mark>  <mark>1889</mark>     <mark>159</mark>     1    <mark>EGRESS_ACL_IPV4</mark>
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

We can see that each ACL applied on egress is counted once per application. It exceeds a single bank capability so it spreads between bank #3 and bank #7.

In summary, we are sharing the ACL on ingress but not on egress. On egress, the entries used in the iTCAM are the multiplication of the ACEs count by the number of times the ACL is applied.

### Unique interface-based ACLs

The scale mentioned earlier (31 ACLs ingress and 255 ACLs egress) can be seen as too restrictive for some specific use-cases. We added the capability to extend this scale with a specific hardware profile:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 ?
  dst-addr         destination address, 32 bit qualifier
  dst-port         destination L4 Port, 16 bit qualifier
  enable-capture   Enable ACL based mirroring. Disables ACL logging
  enable-set-ttl   Enable Setting TTL field
  frag-bit         fragment-bit, 1 bit qualifier
  <mark>interface-based  Enable non-shared interface based ACL</mark>
  location         Location of format access-list ipv4 config
  packet-length    packet length, 10 bit qualifier
  port-range       ipv4 port range qualifier, 24 bit qualifier
  precedence       precedence/dscp, 8 bit qualifier
  proto            protocol type, 8 bit qualifier
  src-addr         source address, 32 bit qualifier
  src-port         source L4 port, 16 bit qualifier
  tcp-flags        tcp-flags, 6 bit qualifier
  ttl-match        Enable matching on TTL field
  udf1             user defined filter
  udf2             user defined filter
  udf3             user defined filter
  udf4             user defined filter
  udf5             user defined filter
  udf6             user defined filter
  udf7             user defined filter
  udf8             user defined filter

RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 interface-based

In order to activate/deactivate this ipv4 profile, you must manually reload the chassis/all line cards
RP/0/RP0/CPU0:5500-6.3.2(config)#
</code>
</pre>
</div>

You can more specific on the key format of these ACLs:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 src-addr src-port dst-addr dst-port interface-based
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv6 src-addr dst-addr dst-port interface-based
</code>
</pre>
</div>

With this approach, the limitations of 31 and 255 respectively are removed. You can configure many more ACLs with fewer ACEs list.


## Statistics

Counters being a precious resource on DNX chipset, the permit entries are not counted by default.

It's possible to change this behavior and enable the statistics on the permit entries in ingress via a specific hw-module profile. As often with hardware profiles, it requires a reload of the chassis or at least the line cards where the features will be used.

This feature is particularly useful if you use ABF and needs to track the flows handled by each ACE.

Note: this profile will not activate counters for egress permits.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile stats ?
  acl-permit    Enable ACL permit stats.
  qos-enhanced  Enable enhanced QoS stats.
RP/0/RP0/CPU0:NCS5500-6.3.2(config)# hw-module profile stats acl-permit

In order to activate/deactivate this stats profile, you must manually reload the chassis/all line cards
RP/0/RP0/CPU0:NCS5500-6.3.2(config)# commit
RP/0/RP0/CPU0:NCS5500-6.3.2(config)# end
RP/0/RP0/CPU0:router# reload location all

Proceed with reload? [confirm]
</code>
</pre>
</div>

Let's take a look at the statistic database allocation before the activation of the profile and what is the difference after the activation and reload:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#show controllers npu resources stats instance 0 loc 0/0/CPU0

System information for NPU 0:
  Counter processor configuration profile: Default
  Next available counter processor:        4

Counter processor: 0                        | Counter processor: 1
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Trap                        95     300  |     Trap                        95     300
    <mark>Policer (QoS)</mark>                <mark>0    6976</mark>  |     <mark>Policer (QoS)</mark>                <mark>0    6976</mark>
    <mark>ACL RX, LPTS</mark>               <mark>148     915</mark>  |     <mark>ACL RX, LPTS</mark>               <mark>148     915</mark>
                                            |
                                            |
Counter processor: 2                        | Counter processor: 3
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    VOQ                         29    8191  |     VOQ                         29    8191
                                            |
                                            |
Counter processor: 4                        | Counter processor: 5
  State: Free                               |   State: Free
                                            |
                                            |
Counter processor: 6                        | Counter processor: 7
  State: Free                               |   State: Free
                                            |
                                            |
Counter processor: 8                        | Counter processor: 9
  State: Free                               |   State: Free
                                            |
                                            |
Counter processor: 10                       | Counter processor: 11
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    L3 RX                        0    8191  |     L3 RX                        0    8191
    L2 RX                        0    8192  |     L2 RX                        0    8192
                                            |
                                            |
Counter processor: 12                       | Counter processor: 13
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 0   16383  |     Interface TX                 0   16383
                                            |
                                            |
Counter processor: 14                       | Counter processor: 15
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 0   16384  |     Interface TX                 0   16384
                                            |
                                            |
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#conf
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config)#hw-module profile stats acl-permit

In order to activate/deactivate this stats profile, you must manually reload the chassis/all line cards
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config)#commit
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config)#end
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#admin

root connected from 127.0.0.1 using console on NCS55A1-24H-6.3.2
sysadmin-vm:0_RP0# reload rack 0

Reload node ? [no,yes] yes
result Rack graceful reload request on 0 acknowledged.

-=|RELOAD|=-----=|RELOAD|=-----=|RELOAD|=-----=|RELOAD|=-----=|RELOAD|=-----=|RELOAD|=-

RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#show controllers npu resources stats instance 0 loc 0/0/CPU0

System information for NPU 0:
  Counter processor configuration profile: ACL Permit
  Next available counter processor:        4

Counter processor: 0                        | Counter processor: 1
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Trap                        95     300  |     Trap                        95     300
    <mark>ACL RX, LPTS</mark>               <mark>147    7891</mark>  |     <mark>ACL RX, LPTS</mark>               <mark>147    7891</mark>
                                            |
                                            |
Counter processor: 2                        | Counter processor: 3
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    VOQ                         29    8191  |     VOQ                         29    8191
                                            |
                                            |
Counter processor: 4                        | Counter processor: 5
  State: Free                               |   State: Free
                                            |
                                            |
Counter processor: 6                        | Counter processor: 7
  State: Free                               |   State: Free
                                            |
                                            |
Counter processor: 8                        | Counter processor: 9
  State: Free                               |   State: Free
                                            |
                                            |
Counter processor: 10                       | Counter processor: 11
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    L3 RX                        0    8191  |     L3 RX                        0    8191
    L2 RX                        0    8192  |     L2 RX                        0    8192
                                            |
                                            |
Counter processor: 12                       | Counter processor: 13
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 0   16383  |     Interface TX                 0   16383
                                            |
                                            |
Counter processor: 14                       | Counter processor: 15
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 0   16383  |     Interface TX                 0   16383
                                            |
                                            |
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#
</code>
</pre>
</div>

Please note that enabling this profile removed the allocation of counters for QoS. It's not possible to count QoS with it. Hw-profiles are always a trade-off.


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

### ACL copy

We support the copy of an ACL to another one. It could be a very useful feature for the operator.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#copy access-list ?
  ethernet-service  Copy Ethernet Service access list
  ipv4              Copy IPv4 access list
  ipv6              Copy IPv6 access list
RP/0/RP0/CPU0:5500-6.3.2#copy access-list ipv4 test-range-24 new-acl
RP/0/RP0/CPU0:5500-6.3.2#
</code>
</pre>
</div>

### DPA

In all the NCS5500 products, we use an abstraction layer between the IOS XR and the hardware. For the operator perspective, this function can be represented by the DPA (Data Plane Abstraction). When a route, a next-hop is added or removed, it goes through the DPA. It's also true for ACLs.

The following show command is used to monitor the number of operation and current status.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2#sh dpa resources ipacl loc 0/7/CPU0

"ipacl" DPA Table (Id: 60, Scope: Non-Global)
--------------------------------------------------
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          <mark>In Use: 1051</mark>            0               0               0
                 Create Requests
                           Total: 4761            0               0               0
                         Success: 4761            0               0               0
                 Delete Requests
                           Total: 3710            0               0               0
                         Success: 3710            0               0               0
                 Update Requests
                           Total: 990             0               0               0
                         Success: 990             0               0               0
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     <mark>HW Failures: 0</mark>               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0

RP/0/RP0/CPU0:5500-6.3.2#sh dpa resources ip6acl loc 0/7/CPU0

"ip6acl" DPA Table (Id: 61, Scope: Non-Global)
--------------------------------------------------
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: 0               0               0               0
                 Create Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                 Delete Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                 Update Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0

RP/0/RP0/CPU0:5500-6.3.2#
</code>
</pre>
</div>

The HW Failures counter is important since it will increment when the software tries to push more entries than what the hardware actually supports.

### hw-module profiles

Several hw-profiles exist to enable specific functions around ACLs. They need a reload to be activated on the system or the line cards after the configuration.

- Enable an IPv4 egress ACL on BVI

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)# hw-module profile acl egress layer3 interface-based
</code>
</pre>
</div>

- Enable permit statistics

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)# hw-module profile stats acl-permit
</code>
</pre>
</div>

- Match on TTL field

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 src-addr src-port enable-set-ttl ttl-match
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 dst-addr dst-port enable-set-ttl ttl-match 
</code>
</pre>
</div>

- Enable the interface-based unique ACL mode

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv4 interface-based
RP/0/RP0/CPU0:5500-6.3.2(config)#hw-module profile tcam format access-list ipv6 src-addr dst-addr dst-port interface-based
</code>
</pre>
</div>


## Resources

CCO guides: Implementing ACLs

[https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/63x/b-ip-addresses-configuration-guide-ncs5500-63x/b-ip-addresses-configuration-guide-ncs5500-63x_chapter_010.html](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/63x/b-ip-addresses-configuration-guide-ncs5500-63x/b-ip-addresses-configuration-guide-ncs5500-63x_chapter_010.html)

CCO guide: ACL commands

[https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html](https://www.cisco.com/c/en/us/td/docs/iosxr/ncs5500/ip-addresses/b-ip-addresses-cr-ncs5500/b-ncs5500-ip-addresses-cli-reference_chapter_01.html)

## Thanks :)

Thanks a lot to Puneet Kalra, Jeff Tayler and Ashok Kumar for their precious help.
