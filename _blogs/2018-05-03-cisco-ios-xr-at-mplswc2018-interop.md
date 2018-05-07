---
published: true
date: '2018-05-03 08:29 -0700'
title: >-
  MPLS + SDN + NFV World @ Paris2018 – Cisco IOS XR participation at
  Interoperability Showcase
author: Jose Liste
tags:
  - iosxr
  - cisco
position: hidden
excerpt: test excerpt
---

{% include toc icon="table" title="Cisco IOS XR at MPLS WC 2018 interop" %}
{% include base_path %}

## Not just another interop

I have represented Cisco for more than 8 years at EANTC-sponsored interoperability events. Going back to 2008, I have experienced more pleasant Berlin winters than I could have ever imagined.

But seriously and in retrospective, it is fascinating to have witnessed several technology (and even political) battles and transitions that shaped the networking industry. Many technologies and entire companies have had short and unsuccessful lives. In my opinion, during the late 2000s our industry was distracted and wasted brain / development cycles; remember PBT, PBB-TE, T-MPLS, MPLS-TP, OpenFlow?

However, one technology has always remained strong – and that is MPLS. During the hype days of OpenFlow, several so-called experts questioned the future of MPLS in operator networks.  At that time, Cisco was starting the journey with Segment Routing (SR). In November of 2012, Cisco Fellow Clarence Filsfils first disclosed the SR concept to network operators. And just one year later, a new working group was officially formed at the IETF - Source Packet Routing In Networking (SPRING). And in 2015, EANTC was already conducting the first public SR interop event. Fast-forward to today and this interop provides the latest proof-point of overwhelming multi-vendor support for SR.

Another technology with a successful trajectory is Ethernet VPN (EVPN). Initially, it was positioned as the next generation solution for Layer 2 VPNs and in particular for Layer 2 Data Center Interconnection (DCI) and multipoint Ethernet LAN (E-LAN) services. From there, it quickly evolved to accommodate other usecases such as point-to-point Ethernet Line (E-LINE) and point-to-multipoint Ethernet Tree (E-Tree) services. Currently, EVPN usecases cross beyond Carrier Ethernet and are deeply entrenched into the datacenter for both intra-subnet and inter-subnet forwarding. In 2013, I was part of the first public interop test of PBB-EVPN at EANTC. In the last five years, multi-vendor support and sophistication of test scenarios have been growing steadily.

But enough of the history lesson!!! Let me move onto the important part of this blog

My goal is to provide a technical overview of Cisco’s participation at this year’s interop showcase with platforms powered by the IOS XR operating system. From a technology perspective, my focus will be on SR and EVPN. Be aware that Cisco was also represented by our Datacenter product line and the Network Services Orchestrator (NSO).

First, I strongly recommend the reader to go over [EANTC’s official public whitepaper](http://www.eantc.de/en/showcases/mpls_sdn_2018). Complementing the information on their report, I take a step back hoping to provide further context and perspective of the results – Why should I CARE ABOUT the event? What do these results REALLY represent? Then last, I provide further insight as to what operators should keep in mind when evaluating vendor offerings – What ELSE to keep in mind beyond the results in the report?
