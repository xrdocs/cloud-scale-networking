---
published: true
date: '2018-07-20 16:25 +0200'
title: 'Security ACL on NCS5500 (Part2): Hybrid ACL'
author: Nicolas Fevrier
excerpt: >-
  Second part on the NCS5500 Access-lists. This time, we focus on the Hybrid
  ACL.
position: top
tags:
  - iosxr
  - acl
  - ncs5500
  - hybrid
  - scale
  - compressed
  - xr
---
{% include toc icon="table" title="NCS5500 Security Access-lists - Part2: Hybrid ACL" %} 

You can find more content related to NCS5500 including routing memory management, VRF, URPF, Netflow following this [link](https://xrdocs.io/cloud-scale-networking/tutorials/).

## Introduction

After introducing [the traditional ACLs in a recent post](https://xrdocs.io/cloud-scale-networking/tutorials/security-acl-on-ncs5500-part1/), let's focus on an interesting approach offering significantly higher scale and better flexibility: the hybrid ACLs.

Let's start with a 5-minute video to illustrate this feature and its benefits:

[![NCS5500 Hybrid ACLs](https://img.youtube.com/vi/xIUgbL7d6tk/0.jpg)](https://www.youtube.com/watch?v=xIUgbL7d6tk){: .align-center}

[https://www.youtube.com/watch?v=xIUgbL7d6tk](https://www.youtube.com/watch?v=xIUgbL7d6tk)

## Support

Hybrid ACL feature uses two databases to store the information:
- internal TCAM (present inside the NPU)
- external TCAM

That implies, only the systems equiped with external TCAM can be used here. It's true for Qumran-MX, Jericho and Jericho+ based routers and line cards.

![image-center]({{site.baseurl}}/images/-SE.png){: .align-center}

Hybrid ACLs can be used with IPv4 and IPv6 in ingress direction. So, L2 ACLs or L3 ICMP are not in the scope of this feature. It can not be applied in egress.

## Configuration

Before jumping into the configuration aspects, let's define the concept of object-groups: it's a structure of data used to describe elements and called later in an access-list entry line (permit or deny).  
We will use two types of object-groups:
- network object-groups: set of network addresses
- port object-groups: set of UDP or TCP ports

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
RP/0/RP0/CPU0:TME-5508-1-6.3.2#conf

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group ?
  network  Network object group
  port     Port object group

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ?
  ipv4  IPv4 object group
  ipv6  IPv6 object group

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ipv4 TEST ?
  A.B.C.D/length  IPv4 address/prefix
  description     Description for the object group
  host            A single host address
  object-group    Nested object group
  range           Range of host addresses

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ipv6 TESTv6 ?
  X : X : : X/length  IPv6 prefix x : x : : x/y
  description    Description for the object group
  host           A single host address
  object-group   nested object group
  range          Range of host addresses

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group port TEST ?
  description   description for the object group
  eq            Match packets on ports equal to entered port number
  gt            Match packets on ports greater than entered port number
  lt            Match packets on ports less than entered port number
  neq           Match packets on ports not equal to entered port number
  object-group  nested object group
  range         Match only packets on a given port range

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#
</code>
</pre>
</div>

As you can see in the help options of the IOS XR configuration:
- we use specific object-groups per address-family IPv4 and IPv6
- you can describe host, networks + prefix length or ranges of addresses
- port object-groups are not specifying if they are for UDP or TCP, it will be described in the ACE and can potentially be re-used for both
- you can describe ports with logical "predicates" like greater than, less than, equal or even ranges

For example:

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
object-group network ipv4 OBJ-DNS-network
 183.13.48.0/24
 host 183.13.64.23
 range 183.13.64.123 183.13.64.150
!
object-group port OBJ-Email-Ports
 eq smtp
 eq pop3
 eq 143
 eq 443
</code>
</pre>
</div>

These objects will be used in an access-list entry to describe the flow(s).

Example1: IPv4 addresses, source and destination

![image-center]({{site.baseurl}}/images/acl2.png){: .align-center}

Example2: IPv4 addresses and ports, source and destination

![image-center]({{site.baseurl}}/images/acl.png){: .align-center}

Separating addresses and ports in two groups and calling these objects in access-list entries line offers an unique flexibility.  
It's easy to create a matrix that would take dozens or even hundreds of lines if they were described one by one with traditional ACLs.

Let's imagine a case with email servers: 17 servers or networks and 8 ports.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run object-group network ipv4 OBJ-Email-Nets
object-group network ipv4 OBJ-Email-Nets
 183.13.64.23/32
 183.13.64.38/32
 183.13.64.39/32
 183.13.64.48/32
 183.13.64.133/32
 183.13.64.145/32
 183.13.64.146/32
 183.13.64.155/32
 183.13.65.0/24
 183.13.65.128/25
 183.13.66.15/32
 183.13.66.17/32
 183.13.66.111/32
 183.13.66.112/32
 183.13.68.0/23
 192.168.1.0/24
 195.14.52.0/24
!
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run object-group network ipv4 OBJ-Email-Nets | i "/" | utility wc -l
17
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run object-group port OBJ-Email-Ports
object-group port OBJ-Email-Ports
 eq smtp
 eq www
 eq pop3
 eq 143
 eq 443
 eq 445
 eq 993
 eq 995
!
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run object-group port OBJ-Email-Ports | i "eq" | utility wc -l
8
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run ipv4 access-list FILTER-IN
ipv4 access-list FILTER-IN
 10 remark Email Servers
 20 permit tcp any net-group OBJ-Email-Nets port-group OBJ-Email-Ports
!
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run int hu 0/7/0/2
interface HundredGigE0/7/0/2
 ipv4 address 27.7.4.1 255.255.255.0
 ipv6 address 2001:27:7:4::1/64
 load-interval 30
 ipv4 access-group FILTER-IN ingress compress level 3
!
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

You could visualize what would be the traditional/flat ACL with this show command:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists ipv4 FILTER-IN <mark>expanded</mark>
ipv4 access-list FILTER-IN
 20 permit tcp any 183.13.68.0 0.0.1.255 eq smtp
 20 permit tcp any 183.13.68.0 0.0.1.255 eq www
 20 permit tcp any 183.13.68.0 0.0.1.255 eq pop3
 20 permit tcp any 183.13.68.0 0.0.1.255 eq 143
 20 permit tcp any 183.13.68.0 0.0.1.255 eq 443
 20 permit tcp any 183.13.68.0 0.0.1.255 eq 445
 20 permit tcp any 183.13.68.0 0.0.1.255 eq 993
 20 permit tcp any 183.13.68.0 0.0.1.255 eq 995
 20 permit tcp any 192.168.1.0 0.0.0.255 eq smtp
 20 permit tcp any 192.168.1.0 0.0.0.255 eq www
 20 permit tcp any 192.168.1.0 0.0.0.255 eq pop3
 20 permit tcp any 192.168.1.0 0.0.0.255 eq 143
 20 permit tcp any 192.168.1.0 0.0.0.255 eq 443
 ...
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists ipv4 FILTER-IN expanded | utility wc -l
ipv4 access-list FILTER-IN
136
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

This line of access-list FILTER-IN equals to a matrix of 136 entries:

![image-center]({{site.baseurl}}/images/matrix-18x7.png){: .align-center}

To add one new mail server, it's easy:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#conf
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ipv4 OBJ-Email-Nets
RP/0/RP0/CPU0:TME-5508-1-6.(config-object-group-ipv4)#183.13.64.157/32
RP/0/RP0/CPU0:TME-5508-1-6.(config-object-group-ipv4)#commit
RP/0/RP0/CPU0:TME-5508-1-6.(config-object-group-ipv4)#end
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists ipv4 FILTER-IN expanded | i .157
ipv4 access-list FILTER-IN
 20 permit tcp any host 183.13.64.157 eq smtp
 20 permit tcp any host 183.13.64.157 eq www
 20 permit tcp any host 183.13.64.157 eq pop3
 20 permit tcp any host 183.13.64.157 eq 143
 20 permit tcp any host 183.13.64.157 eq 443
 20 permit tcp any host 183.13.64.157 eq 445
 20 permit tcp any host 183.13.64.157 eq 993
 20 permit tcp any host 183.13.64.157 eq 995
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

Adding one line in the object group for networks, it's like we add 8 lines in a flat ACL.  
As you can see, it's a very flexible way to manage your access-lists.

## Operation

The hybrid ACL implies we defined object-groups members but also that we used compression:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh run int hu 0/7/0/2
interface HundredGigE0/7/0/2
 ipv4 address 27.7.4.1 255.255.255.0
 ipv6 address 2001:27:7:4::1/64
 load-interval 30
 ipv4 access-group FILTER-IN ingress <mark>compress level 3</mark>
!
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

Once the ACL has been applied in ingress and with compression level 3 on the interface, the PMF block of the pipeline will perform two look-ups:
- the first one in the external TCAM for objects (compressed result of source address, destination address and source port)
- the second one in the internal TCAM for the ACL with destination port and the compressed result of the first lookup

![image-center]({{site.baseurl}}/images/2step.png){: .align-center}

It's not necessary to remove the ACL from the interface to edit the content, it can be done "in-place" and without traffic impact.

### Coexistence with flat ACL

It's possible to use traditional access-list entries in the same ACL. Each line is pretty much independant from the others, with or without object-groups:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
ipv4 access-list FILTER-IN
 10 permit tcp any net-group SRC port-group PRT
 20 permit tcp net-group SRC any port-group PRT
 30 permit tcp net-group SRC port-group PRT any
 40 permit tcp any port-group PRT net-group SRC
 50 permit udp any host 1.2.3.3 eq 445
 60 permit ipv4 1.3.5.0/24 any
 70 permit ipv4 any 1.3.5.0/24
!
</code>
</pre>
</div>

## eTCAM Carving requirement

A portion of the eTCAM should be used to store a part ACL information (compressed or not).

### Jericho+ systems

For systems based on Jericho+, we don't have anything to worry about: the eTCAM can handle this information without any specific configuration or preparation

### Jericho systems with 6.1.x and 6.3.2 onwards

For systems based on Jericho running 6.1.x and 6.3.2 onwards: no carving done by default, it will be necessary to change the configuration before enabling hybrid ACLs

![image-center]({{site.baseurl}}/images/632.png){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-6.3.2#sh contr npu ext loc 0/7/CPU0

External TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0      80b    FLP         2047993  7       15   IPV4 DC
1    0      80b    FLP         2047993  7       15   IPV4 DC
2    0      80b    FLP         2047993  7       15   IPV4 DC
3    0      80b    FLP         2047993  7       15   IPV4 DC

RP/0/RP0/CPU0:NCS5508-6.3.2#
</code>
</pre>
</div>

If you try to apply a compressed (level 3) ACL on this interface 0/7/0/2 (LC0/7 is a 24x100-SE), the system will refuse the commit and the show config failed will explain it can not be done on this line card:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#int hu 0/7/0/2
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config-if)# ipv4 access-group FILTER-IN ingress compress level 3
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config-if)#commit

% Failed to commit one or more configuration items during a pseudo-atomic operation. All changes made have been reverted. Please issue 'show configuration failed [inheritance]' from this session to view the errors
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config-if)#show configuration failed

!! SEMANTIC ERRORS: This configuration was rejected by
!! the system due to semantic errors. The individual
!! errors with each failed configuration command can be
!! found below.

interface HundredGigE0/7/0/2
 ipv4 access-group FILTER-IN ingress compress level 3
!!% 'dnx_feat_mgr' detected the 'resource not available' condition 'ACL compression is not supported on this LC due to configuration or LC type'
!
end

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config-if)#
</code>
</pre>
</div>

We took the decision starting from 6.3.2 onwards to let the carving decision to the user and no longer taking arbitrary 20% of the resource whether or not the operator decided to use this feature.

A manual carving of the external TCAM is necessary and can be done with the following steps:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#hw-module profile tcam acl-prefix percent 20
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#commit

RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#end
RP/0/RP0/CPU0:TME-5508-1-6.3.2#admin

root connected from 127.0.0.1 using console on TME-5508-1-6.3.2
sysadmin-vm:0_RP0# reload rack 0

Reload node ? [no,yes] yes
result Rack graceful reload request on 0 acknowledged.
sysadmin-vm:0_RP0#
</code>
</pre>
</div>

Once reload is completed, you can check the external TCAM carving, it's now ready for hybrid ACLs.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh contr npu ext loc 0/7/CPU0
Sat Jul 21 13:02:35.879 PDT
External TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0      80b    FLP         886729   751671  15   IPV4 DC
0    1      80b    FLP         8192     0       81   <mark>INGRESS_IPV4_SRC_IP_EXT</mark>
0    2      80b    FLP         8192     0       82   <mark>INGRESS_IPV4_DST_IP_EXT</mark>
0    3      160b   FLP         8192     0       83   <mark>INGRESS_IPV6_SRC_IP_EXT</mark>
0    4      160b   FLP         8192     0       84   <mark>INGRESS_IPV6_DST_IP_EXT</mark>
0    5      80b    FLP         8192     0       85   <mark>INGRESS_IP_SRC_PORT_EXT</mark>
0    6      80b    FLP         8192     0       86   <mark>INGRESS_IPV6_SRC_PORT_EXT</mark>
...
</code>
</pre>
</div>

### Jericho systems with 6.2.x and 6.3.1/6.3.15

For systems based on Jericho running 6.2.x and 6.3.1/6.3.1, 20% of the eTCAM is pre-allocated, even if you don't plan to use hybrid ACLs.

Nothing should be done if we decide to enable this feature.

![image-center]({{site.baseurl}}/images/62x.png){: .align-center}

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-6.2.3#sh contr npu externaltcam loc 0/7/CPU0

External TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0      80b    FLP         498950   1139450  15   IPV4 DC
0    1      80b    FLP         28672    0       76   INGRESS_IPV4_SRC_IP_EXT
0    2      80b    FLP         28672    0       77   INGRESS_IPV4_DST_IP_EXT
0    3      160b   FLP         26624    0       78   INGRESS_IPV6_SRC_IP_EXT
0    4      160b   FLP         26624    0       79   INGRESS_IPV6_DST_IP_EXT
0    5      80b    FLP         28672    0       80   INGRESS_IP_SRC_PORT_EXT
0    6      80b    FLP         28672    0       81   INGRESS_IPV6_SRC_PORT_EXT
...
</code>
</pre>
</div>

Hybrid ACL can be applied directly. No specific preparation needed.

## Monitoring

Let's take a look at several show commands useful to verify the ACL configuration and application:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists ipv4 FILTER-IN object-groups
ACL Name : FILTER-IN
Network Object-group :
    OBJ-Email-Nets
---------------------------
                   Total 1
Port Object-group :
    OBJ-Email-Ports
---------------------------
                   Total 1
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists FILTER-IN usage pfilter loc 0/7/CPU0
Interface : HundredGigE0/7/0/2
    Input  ACL : Common-ACL : N/A  ACL : FILTER-IN  (<mark>comp-lvl 3</mark>)
    Output ACL : N/A
</code>
</pre>
</div>

The verify option will permit to check if the compression happened correctly (expect "passed" in these lines).

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists FILTER-IN hardware ingress interface hundredGigE 0/7/0/2 verify location 0/7/CPU0

Verifying TCAM entries for FILTER-IN
Please wait...



    INTF    NPU lookup  ACL # intf Total  compression Total   result failed(Entry) TCAM entries
                type    ID  shared ACES   prefix-type Entries        ACE SEQ #     verified
 ---------- --- ------- --- ------ ------ ----------- ------- ------ ------------- ------------

HundredGigE0_7_0_2 (ifhandle: 0x3800120)

              0 IPV4      1      1      1 COMPRESSED        9 passed                          9
                                          SRC IP            1 passed                          1
                                          DEST IP          19 passed                         19
                                          SRC PORT          1 passed                          1
</code>
</pre>
</div>

Since the ACL is stored in both internal and external TCAMs, we can check the memory utilization with the following:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#show controllers npu internaltcam loc 0/7/CPU0
Internal TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0\1    320b   pmf-0       1979     48      7    INGRESS_LPTS_IPV4
0    0\1    320b   pmf-0       1979     8       12   INGRESS_RX_ISIS
0    0\1    320b   pmf-0       1979     2       32   INGRESS_QOS_IPV6
0    0\1    320b   pmf-0       1979     2       34   INGRESS_QOS_L2
0    0\1    320b   pmf-0       1979     <mark>9</mark>       49   <mark>INGRESS_HYBRID_ACL</mark>
0    2      160b   pmf-0       2044     2       31   INGRESS_QOS_IPV4
0    2      160b   pmf-0       2044     1       33   INGRESS_QOS_MPLS
0    2      160b   pmf-0       2044     1       42   INGRESS_ACL_L2
0    3      160b   egress_acl  2031     17      4    EGRESS_QOS_MAP
0    4\5    320b   pmf-0       2013     35      8    INGRESS_LPTS_IPV6
0    6      160b   Free        2048     0       0
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
...
RP/0/RP0/CPU0:TME-5508-1-6.3.2#show controllers npu externaltcam loc 0/7/CPU0
External TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0      80b    FLP         886729   751671  15   IPV4 DC
0    1      80b    FLP         8191     <mark>1</mark>       81   <mark>INGRESS_IPV4_SRC_IP_EXT</mark>
0    2      80b    FLP         8173     <mark>19</mark>      82   <mark>INGRESS_IPV4_DST_IP_EXT</mark>
0    3      160b   FLP         8192     0       83   INGRESS_IPV6_SRC_IP_EXT
0    4      160b   FLP         8192     0       84   INGRESS_IPV6_DST_IP_EXT
0    5      80b    FLP         8191     <mark>1</mark>       85   <mark>INGRESS_IP_SRC_PORT_EXT</mark>
0    6      80b    FLP         8192     0       86   INGRESS_IPV6_SRC_PORT_EXT
...
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

The DPA is the abstraction layer used for the programming of the hardware.  
If the resource is exhausted, you'll find "HW Failures" count incremented:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh dpa resources ipaclprefix loc 0/7/CPU0

"ipaclprefix" DPA Table (Id: 111, Scope: Non-Global)
--------------------------------------------------
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: 20              0               0               0
                 Create Requests
                           Total: 20              0               0               0
                         Success: 20              0               0               0
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

RP/0/RP0/CPU0:TME-5508-1-6.3.2#
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh dpa resources scaleacl loc 0/7/CPU0

"scaleacl" DPA Table (Id: 114, Scope: Non-Global)
--------------------------------------------------
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: 9               0               0               0
                 Create Requests
                           Total: 9               0               0               0
                         Success: 9               0               0               0
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

RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

You note the size of each object-group is one more than the number of entries. It's always +1 for IPv4 and +3 for IPv6.
{: .notice--info}

The statistics can be also monitored. In this example, we don't count permit ACLs (profile not enabled by default). Check the ACL entries in both engines.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh contr npu resources stats inst 0 loc 0/7/CPU0

System information for NPU 0:
  Counter processor configuration profile: Default
  Next available counter processor:        4

Counter processor: 0                        | Counter processor: 1
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Trap                        95     300  |     Trap                        95     300
    Policer (QoS)                0    6976  |     Policer (QoS)                0    6976
    ACL RX, LPTS               <mark>182</mark>     <mark>915</mark>  |     ACL RX, LPTS               <mark>182</mark>     <mark>915</mark>
                                            |
                                            |
Counter processor: 2                        | Counter processor: 3
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    VOQ                        140    8191  |     VOQ                        140    8191
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
    L3 RX                        3    8191  |     L3 RX                       11    8191
    L2 RX                        1    8192  |     L2 RX                        1    8192
                                            |
                                            |
Counter processor: 12                       | Counter processor: 13
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 4   16383  |     Interface TX                 6   16383
                                            |
                                            |
Counter processor: 14                       | Counter processor: 15
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 0   16384  |     Interface TX                 0   16384
                                            |
                                            |
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

## Notes

### Object-group and non-eTCAM?

It's possible to define object-based ACLs and apply them in non-eTCAM systems.  
They will be expanded and programmed in the internal TCAM. But it will not be possible to use the compression.

### Use of ranges

Ranges are supported but within a limit of 24 range-IDs.

### Compression

We only support the level 3 compression:
- source address, destination address, source port are compressed and stored in the external TCAM
- destination port is not compressed
- level 0 equals not-compressed

### ACL content edition

It's possible edit the object-groups in-place, without having to remove the acl from the interface. But an edition of netgroup or portgroup will force the replacement and reprogramming of the ACL. The counters will be reset.

### Statistics

Permits are not counted by default. It's necessary to enable another hw-profile to count permit but it will replace the QoS counters:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#conf
hw-module profile stats acl-permitRP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#hw-module profile stats acl-permit
In order to activate/deactivate this stats profile, you must manually reload the chassis/all line cards
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#commit
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#end
RP/0/RP0/CPU0:TME-5508-1-6.3.2#admin

root connected from 127.0.0.1 using console on TME-5508-1-6.3.2
sysadmin-vm:0_RP0# reload rack 0
Sat Jul  21 22:27:14.156 UTC
Reload node ? [no,yes] yes
result Rack graceful reload request on 0 acknowledged.
</code>
</pre>
</div>

After the reload:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh contr npu resources stats inst 0 loc 0/7/CPU0

System information for NPU 0:
  Counter processor configuration profile: ACL Permit
  Next available counter processor:        4

Counter processor: 0                        | Counter processor: 1
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Trap                        95     300  |     Trap                        95     300
    ACL RX, LPTS               <mark>182</mark>    <mark>7891</mark>  |     ACL RX, LPTS               <mark>182</mark>    <mark>7891</mark>
                                            |
                                            |
Counter processor: 2                        | Counter processor: 3
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    VOQ                        140    8191  |     VOQ                        140    8191
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
    L3 RX                        3    8191  |     L3 RX                        9    8191
    L2 RX                        1    8192  |     L2 RX                        1    8192
                                            |
                                            |
Counter processor: 12                       | Counter processor: 13
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 4   16383  |     Interface TX                 6   16383
                                            |
                                            |
Counter processor: 14                       | Counter processor: 15
  State: In use                             |   State: In use
                                            |
  Application:              In use   Total  |   Application:              In use   Total
    Interface TX                 0   16383  |     Interface TX                 0   16383
                                            |
                                            |
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

Now permit matches will be counted:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh access-lists ipv4 FILTER-IN hardware ingress interface hundredGigE 0/7/0/2 loc 0/7/CPU0
ipv4 access-list FILTER-IN
 10 permit tcp any net-group SRC port-group PRT
 20 permit tcp net-group SRC any port-group PRT (<mark>3 matches</mark>)
 30 permit tcp net-group SRC port-group PRT any
 40 permit tcp any port-group PRT net-group SRC
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

If packets are matching a permit entry in the ACL and are targeted to the router, they will be punted but not counted in the ACL "matches".
{: .notice--info}

## Conclusion

We hope we demonstrated the power of hybrid ACL for infrastructure security. They offer a lot of flexibility and huge scale.  
Definitely something you should consider for a greenfield deployment.

Nevertheless, moving from existing traditional ACLs is not an easy task.  It's common to see networks with very large flat ACLs, poorly documented.  The operators are usually very uncomfortable touching it.

In these brownfield scenarios, it's mandatory to start an entire project to redefine the flows that are allowed and forbidden through these routers and it could be a long process.

Post Scriptum: the address ranges used in this article (183.13.x.y) are random numbers I picked. We are not sharing any real ACL from any real operator here. We could have picked the 192.0.2.0/24 but it would have made the examples less relevant.
{: .notice--success}
