---
published: false
date: '2018-07-20 16:25 +0200'
title: 'Security ACL on NCS5500 (Part2): Hybrid ACL'
author: Nicolas Fevrier
excerpt: >-
  Second part on the NCS5500 Access-lists. This time, we focus on the Hybrib
  ACL.
position: top
---
{% include toc icon="table" title="NCS5500 Security Access-lists - Part2: Hybrid ACL" %} 

## Introduction

After introducing [the traditional ACLs in a recent post](https://xrdocs.io/cloud-scale-networking/tutorials/security-acl-on-ncs5500-part1/), let's focus on an interesting approach offering significantly higher scale and better flexibility: the hybrid ACLs.

Let's start with a 5-minute video to illustrate this features and its benefits:

[![NCS5500 Hybrid ACLs](https://img.youtube.com/vi/xIUgbL7d6tk/0.jpg](https://www.youtube.com/watch?v=xIUgbL7d6tk){: .align-center}

[https://www.youtube.com/watch?v=xIUgbL7d6tk](https://www.youtube.com/watch?v=xIUgbL7d6tk)

## Support

Hybrid ACL feature uses two databases to store the information:
- internal TCAM (present inside the NPU)
- external TCAM

That implies, only the systems equiped with external TCAM can be used for this feature. It's true for Qumran-MX, Jericho and Jericho+ based routers and line cards.

![-SE.png]({{site.baseurl}}/images/-SE.png)

Hybrid ACLs can be used with IPv4 and IPv6 in ingress direction. So, L2 ACLs or L3 ICMP are not in the scope of this feature. They can noy be applied in egress.

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
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ipv4 ?
  netobj1         IPv4 object group name - maximum 64 characters
  OBJ-50-net      IPv4 object group name - maximum 64 characters
  OBJ-500-net     IPv4 object group name - maximum 64 characters
  OBJ-Email-Nets  IPv4 object group name - maximum 64 characters
  WORD            IPv4 object group name - maximum 64 characters
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ipv4 TEST ?
  A.B.C.D/length  IPv4 address/prefix
  description     Description for the object group
  host            A single host address
  object-group    Nested object group
  range           Range of host addresses
  <cr>
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group network ipv6 TESTv6 ?
  X:X::X/length  IPv6 prefix x:x::x/y
  description    Description for the object group
  host           A single host address
  object-group   nested object group
  range          Range of host addresses
  <cr>
RP/0/RP0/CPU0:TME-5508-1-6.3.2(config)#object-group port TEST ?
  description   description for the object group
  eq            Match packets on ports equal to entered port number
  gt            Match packets on ports greater than entered port number
  lt            Match packets on ports less than entered port number
  neq           Match packets on ports not equal to entered port number
  object-group  nested object group
  range         Match only packets on a given port range
  <cr>
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
 eq www
 eq pop3
 eq 143
 eq 443
 eq 445
 eq 993
 eq 995
!
</code>
</pre>
</div>

These objects will be used in an access-list entry to describe.

Example1: IPv4 addresses, source and destination

![acl2.png]({{site.baseurl}}/images/acl2.png)

Example2: IPv4 addresses and ports, source and destination

![acl.png]({{site.baseurl}}/images/acl.png)

## eTCAM Carving requirement

A portion of the eTCAM should be used to store a port ACL information (compressed or not).

For systems based on Jericho+, we don't have anything to worry about: the eTCAM can handle this information without any specific configuration.

For systems based on Jericho, it will depend on the IOS XR release:

- for 6.1.x and 6.3.2 onwards, no carving done by default, it will be necessary to change the configuration before enabling hybrid ACLs

![632.png]({{site.baseurl}}/images/632.png)

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508-6.3.2#sh contr npu ext loc 0/6/CPU0

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



- for 6.2.x and 6.3.1/6.3.1, 20% of the eTCAM is pre-allocated, even if you use hybrid. Nothing should be done if we decide to enable this feature.

![62x.png]({{site.baseurl}}/images/62x.png)

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-6.2.3#sh contr npu externaltcam loc 0/6/CPU0

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











<div class="highlighter-rouge">
<pre class="highlight">
<code>

</code>
</pre>
</div>



