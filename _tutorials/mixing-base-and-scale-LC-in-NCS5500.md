---
published: true
date: '2018-05-31 08:47 +0000'
title: Mixing Base and Scale Line Cards in the same Chassis
author: Nicolas Fevrier
excerpt: >-
  Introduction of the "Selective Route Download" feature allowing the mix of
  Base and Scale line cards in the same NCS5500 Chassis.
tags:
  - ncs5500
  - scale
  - base
  - chassis
  - ios xr
  - xr
  - SRD
Position: top
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E09 Mixing Base and Scale Line Cards in the same Chassis

### Previously on "Understanding NCS5500 Resources"

In previous posts... We detailed how routes are stored in systems with or without external TCAM, based on  Jericho or Jericho+ forwarding ASICs, with URPF, in VRFs, ...

You can check all the former articles [in the page here](https://xrdocs.github.io/cloud-scale-networking/tutorials/).

Let's address today a recurring question: "What happens if you mix different types of line cards in a chassis?" and we will add the question people should ask more often: "Where do I position the Base line cards and the Scale line cards in my network design?".

We have today 7 different line card types offering a variety of services:
- MACsec encryption
- grey or colored (IPoDWDM) ports
- 40G or 40G/100G capable ports
- with different routing scales (Jericho, Jericho with eTCAM, Jericho+ with eTCAM)
and soon we will see even more diversity with the introduction of the MOD line cards:
- different framing capability (LAN Phy, WAN Phy, OTN)
- different colored optics (ACO or DCO)
- and 25G SFP28 native ports or 4x25G breakout

![Chassis.jpg]({{site.baseurl}}/images/Chassis.jpg){: .align-center}

Below, a 8min video covering this topic.

[https://www.youtube.com/watch?v=FEDFNyuBj3g](https://www.youtube.com/watch?v=FEDFNyuBj3g)

[![Youtube Video](https://img.youtube.com/vi/FEDFNyuBj3g/0.jpg)](https://www.youtube.com/watch?v=FEDFNyuBj3g){: .align-center}


### Selective Route/FIB Download

First, let's clarify the following doubts:
- no, we are not limiting the route scale at the level of the lowest common denominator: we will use specific features to organize the route distribution between Base and Scale line cards.
- no, the system is not breaking nor the packets are punted to the RP CPU or LC CPU when we exceed the limit of a given memory type: the level of abstraction (DPA) used between the routing process and the hardware FIB is controlling all the resources and will refuse to push more entries if a database is full.

Now, let's introduce this feature used to granularily decide where the routes should be populated.

It's named "Selective Route Download", the name is self-descriptive :) We will use it to decide where a given route is meant to be programmed: in the Base line cards, in the Scale line cards, or both.

It will follow some simple principles:
- all the IGP routes (dynamic like ISIS or OSPF, or static/connected) will be programmed in all types of line cards
- by default, BGP routes are programmed in all line card types. "Default" here implies the operator didn't do anything special in the configuration
- BGP paths can be colored as "external-reach-only" and they will be only programmed in -SE line cards (with an external TCAM). The coloring is defined locally via configuration.

![external-reach.jpg]({{site.baseurl}}/images/external-reach.jpg){: .align-center}


### Configuration examples

We can decide to mark these BGP paths in multiple ways. Here are two configurations example, but it's not limited to them.

In the first example, we want to mark "external-reach" all BGP routes received from a specific peer.
We will proceed in three steps:
1 - he routes received from this peer with a specific community 100:111
2 - then we will match this particular community to set the path-color external-reach
3 - finally, we call this policy in a table-policy.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
route-policy PEER-EXT
  set community PEER-EXT-comm
end-policy
!
community-set PEER-EXT-comm
 100:111
end-set
!
route-policy HILO-FIB
  if community matches-any PEER-EXT-comm then
    set path-color external-reach
    pass
  else
    pass
  endif
end-policy
!
router bgp 100
address-family ipv4 unicast
  table-policy HILO-FIB
!
neighbor 192.168.100.151
  address-family ipv4 unicast
   route-policy PEER-EXT in
   maximum-prefix 8000000 75
   route-policy PERMIT-ANY out
</code>
</pre>
</div>

In this second example, we don't differentiate the routes from their peer of origin but more from their nature (for instance the prefix length).

<div class="highlighter-rouge">
<pre class="highlight">
<code>
route-policy HILO-FIB
  if destination in (100.0.0.0/8 le 24, 3000::/8 le 64) then
    set path-color external-reach
    pass
  else
    pass
  endif
end-policy
!
router bgp 100
address-family ipv4 unicast
  table-policy HILO-FIB
!
neighbor 192.168.100.151
  address-family ipv4 unicast
   route-policy PEER-EXT in
   maximum-prefix 8000000 75
   route-policy PERMIT-ANY out
</code>
</pre>
</div>

We could easily imagine other situations where the BGP routes would be marked with specific communities by internet border routers and the local router would take reach-only marking decision based on these communities.


### Verification

The route now carries the color as a locally-significant attribute and we can verify it with simple "show route" CLI:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508#sh route 1.0.144.0/20

Routing entry for 1.0.144.0/20
  Known via "bgp 100", distance 200, metric 0, <mark>external-reach-lc-only</mark>
  Tag 2914, type internal
  Installed Nov 27 22:48:56.925 for 00:00:45
  Routing Descriptor Blocks
    192.168.100.151, from 192.168.100.151
      Route metric is 0
  No advertising protos.

RP/0/RP0/CPU0:NCS5508#
RP/0/RP0/CPU0:NCS5508#sh cef 1.0.144.0/20 detail

1.0.144.0/20, version 25081094, <mark>external-reach-lc-only</mark>, internal 0x5000001 0x0 (ptr 0x8f485390) [1], 0x0 (0x0), 0x0 (0x0)
 Updated Nov 27 22:48:56.929
 local adjacency 192.168.100.151
 Prefix Len 20, traffic index 0, precedence n/a, priority 4
  gateway array (0x8e0e9250) reference count 655801, flags 0x2010, source rib (7), 0 backups
                [1 type 3 flags 0x48501 (0x8e18f758) ext 0x0 (0x0)]
  LW-LDI[type=0, refc=0, ptr=0x0, sh-ldi=0x0]
  gateway array update type-time 1 Nov 27 22:48:56.929
 LDI Update time Nov 27 22:48:56.929
   via 192.168.100.151/32, 2 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0x8e0bf1b0 0x0]
    next hop 192.168.100.151/32 via 192.168.100.151/32

    Load distribution: 0 (refcount 1)

    Hash  OK  Interface                 Address
    0     Y   MgmtEth0/RP0/CPU0/0       192.168.100.151

RP/0/RP0/CPU0:NCS5508#
</code>
</pre>
</div>


### Network design considerations

To understand where the different types of line cards should be positioned in a network architecture, it is important to understand how the packet routing decision happens. The NCS5500 behaves differently than traditional IOS XR platforms like CRS, ASR9000 or NCS6000. In these platforms, we used a two-step lookup model where both the ingress and egress line card ASICs/NPUs are involved. Let's take an ASR9000 as example: both the -TR and -SE line cards have the full routing view and the difference between the position of the two is related to specific features like QoS but not linked to the route scale.

In the NCS5500, -SE and non-SE line cards have the same QoS capability but different route scale. The lookup happens mostly in the ingress pipeline of the Jericho/Jericho+ ASIC as shown in this diagram.

![ingress.jpg]({{site.baseurl}}/images/ingress.jpg){: .align-center}

It implies that the ingress blocks need access to the database where are stored the routing information (route, next-hop, load-balancing info, ...).

To decide where should be positioned each type of line card, you have to take the perspective of the packet :) When a packet needs to reach some destination on internet, the ingress line card must have the full view and it's expected it will be the high(er)-scale board.

In the following diagram, we illustrate it with two routers. One is facing the internet (peering role), and the other one is connected to an internet content server (DC role). Both are connected to the core network (full IP or MPLS).

![topo1.jpg]({{site.baseurl}}/images/topo1.jpg){: .align-center}

On the Peering router:
- packets received from the core (left to right) and targeted to the internet require a lookup in the full view, we need a Scale line card here.
- packets received from internet (right to left) and targeted to a  host address simply need to find the internal routes, a Base line card can do this job.

Considering that current DWDM and the MACsec line cards are non-SE, they can be used to reach distant internet exchange point (IXPs) and if needed can be encrypted. But if we want to position such cards for core-facing interfaces, ie. with 100G/200G colored ports and/or MACsec encryption, we will need the MOD-SE line card (planned for the end of calendar year 2018).

On the DC router.
- packets received from the content server (left to right) and targeted to internet will also need the full public view. That's why we advise using a Scale line card here. Alternative options exist to be able to forward packets to some upper layer of the network where the full internet view will be available, or to filter the routes learnt to limit the FIB programming to the entries actually carrying traffic. But they require additional study. Most of the users will position Scale line cards for the sake of simplicity (both from a design and operation perspective).
- packets received from the core (right to left) are targeted to the content server, so the routing table required here is minimal. A Base line card is sufficient.

Another example where CE devices need to reach other CE devides through a MPLS VPN network.

![topo2.jpg]({{site.baseurl}}/images/topo2.jpg){: .align-center}

On both PE routers, ingress interfaces will require Scale line cards since they need to know all the VRF routes plus the internal routes and transport labels. On the core facing interfaces, we will can position non-SE line cards since they only need to store the local/connected CE routes.


### Caveats / Gotchas

- The color marking of the paths is done locally on the router, that implies it should be done on all routers (it's not something advertised over the network through the BGP updates)
- in the case of a link bundle, one port could be on a base line cards which another port could be on a scale line card. In such a situation the parser will refuse to commit the configuration. The aggregated ports must be bundled in the same type of line card (but not necessarily on the same line card)
- It has been asked several times if this feature can be used on fixed-formed factor chassis using Jericho ASIC without eTCAM to filter the routes programmed. It's not a scenario tested/supported and we suggested they use route-filter instead



### Conclusion

Selective Route Download permits a very flexible selection of the BGP prefixes that needs to be programmed only in the -SE line cards. It's easy to configure and operate, and it allows the mix of Base and Scale line cards in the same chassis.