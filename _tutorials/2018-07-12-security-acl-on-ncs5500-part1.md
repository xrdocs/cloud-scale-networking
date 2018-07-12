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

While hybrid-ACL will be only supported on -SE systems with external TCAM, the traditional security ACL can be used on all the systems and line cards of the portfolio. They are available in ingress, egress, for IPv4, IPv6 and L2. We will detail all the supported options later on in this document.

Please note: we don’t cover access-list used for route-filtering in this document. We don’t cover Access-list Based Forwarding feature, or SPAN (packet capture / replication) based on ACL either. We only intend to cover the security ACL aimed at filtering packets going through the routing device. 


## Basic notions on ACLs

An access-list is applied under on interface statement. It contains an address-family, an ACL identifier or name and a direction.

![acl-format.png]({{site.baseurl}}/images/acl-format.png)

An access-list is made of access-list entries (ACEs). Scale both in term of ACL and ACL will depends on the type of interface, the address-family and the direction.

The first part of the ACL defines if it's L2, v4 or v6, and the name then the following lines are representing the Access-list Entries (ACEs).

![ACE-ACL.png]({{site.baseurl}}/images/ACE-ACL.png)

If you don't use numbers to identify the lines when you configure your ACEs, the system will automatically assign multiple of ten and will increment line after line. Then, the operateur will be able to edit the ACL inserting entries using intermediate values or deleting entries with the appropriate value.


## Support (status in IOS XR 6.5.1)

- We support L2 and L3 ACL but “conditions may apply"
- Ingress IPv4 ACLs are supported on L3 physical, bundles, sub-interfaces and bundled sub-interfaces, but also on BVI interfaces.
- Ingress IPv6 ACLs are supported on L3 physical, bundles, sub-interfaces and bundled sub-interfaces, but also on BVI interfaces.
- Egress IPv4 ACLs are supported on L3 physical and bundle interfaces but also on BVI interfaces.
- Egress IPv6 ACLs are supported on L3 physical and bundle interfaces but also on BVI interfaces.
- Egress IPv4 or IPv6 ACLs are NOT supported on L3 sub-interfaces or bundled sub-interfaces
- It’s no possible to apply an L2 ACL on an IPv4/IPv6 (L3) interface.

Let’s summarise:
| Interface Type | Direction | AF | Suppport ? |
|:-----:|:-----:|:-----:|:-----:|
|  L3 Physical | Ingress | IPv4 | xxx |
|  L3 Physical | Ingress | IPv6 | xxx |
|  L3 Physical | Egress | IPv4 | xxx |
|  L3 Physical | Egress | IPv6 | xxx |
|  L3 Bundle | Ingress | IPv4 | xxx |
|  L3 Bundle | Ingress | IPv6 | xxx |
|  L3 Bundle | Egress | IPv4 | xxx |
|  L3 Bundle | Egress | IPv6 | xxx |
|  L3 Sub-interface | Ingress | IPv4 | xxx |
|  L3 Sub-interface | Ingress | IPv6 | xxx |
|  L3 Sub-interface | Egress | IPv4 | xxx |
|  L3 Sub-interface | Egress | IPv6 | xxx |
|  L3 Bundled Sub-interface | Ingress | IPv4 | xxx |
|  L3 Bundled Sub-interface | Ingress | IPv6 | xxx |
|  L3 Bundled Sub-interface | Egress | IPv4 | xxx |
|  L3 Bundled Sub-interface | Egress | IPv6 | xxx |
|  BVI | Ingress | IPv4 | xxx |
|  BVI | Ingress | IPv6 | xxx |
|  BVI | Egress | IPv4 | xxx |
|  BVI | Egress | IPv6 | xxx |


## Scale

The number of ACLs and ACEs we support is expressed per NPU (Qumran-MX, Jericho, Jericho+, …). Since the ACLs are applied on ports, we invite you to check the former blog post describing the port to NPU assignments.

Also, keep in mind that an ACL applied to a bundle interface where the port members span over multiple NPU will see the ACL/ACEs replicated on all the participating NPUs.



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

((( Check memory bank 6 )))



Unique ACL

ACL sharing

Counters
log
frag
ttl
No resequencing

