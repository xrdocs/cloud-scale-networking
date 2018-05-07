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

## Time for another interop

I have represented Cisco for more than 8 years at EANTC-sponsored interoperability events. Going back to 2008, I have experienced more pleasant Berlin winters than I could have ever imagined.

But seriously and in retrospective, it is fascinating to have witnessed several technology (and even political) battles and transitions that shaped the networking industry. Many technologies and entire companies have had short and unsuccessful lives. In my opinion, during the late 2000s our industry was distracted and wasted brain / development cycles; remember PBT, PBB-TE, T-MPLS, MPLS-TP, OpenFlow?

However, one technology has always remained strong – and that is MPLS. During the hype days of OpenFlow, several so-called experts questioned the future of MPLS in operator networks.  At that time, Cisco was starting the journey with Segment Routing (SR). In November of 2012, Cisco Fellow Clarence Filsfils first disclosed the SR concept to network operators. And just one year later, a new working group was officially formed at the IETF - Source Packet Routing In Networking (SPRING). And in 2015, EANTC was already conducting the first public SR interop event. Fast-forward to today and this interop provides the latest proof-point of overwhelming multi-vendor support for SR.

Another technology with a successful trajectory is Ethernet VPN (EVPN). Initially, it was positioned as the next generation solution for Layer 2 VPNs and in particular for Layer 2 Data Center Interconnection (DCI) and multipoint Ethernet LAN (E-LAN) services. From there, it quickly evolved to accommodate other usecases such as point-to-point Ethernet Line (E-LINE) and point-to-multipoint Ethernet Tree (E-Tree) services. Currently, EVPN usecases cross beyond Carrier Ethernet and are deeply entrenched into the datacenter for both intra-subnet and inter-subnet forwarding. In 2013, I was part of the first public interop test of PBB-EVPN at EANTC. In the last five years, multi-vendor support and sophistication of test scenarios have been growing steadily.

But enough of the history lesson!!! Let me move onto the important part of this blog

My goal is to provide a technical overview of Cisco’s participation at this year’s interop showcase with platforms powered by the IOS XR operating system. From a technology perspective, my focus will be on SR and EVPN. Be aware that Cisco was also represented by our Datacenter product line and the Network Services Orchestrator (NSO).

First, I strongly recommend the reader to go over [EANTC’s official public whitepaper](http://www.eantc.de/en/showcases/mpls_sdn_2018). Complementing the information on their report, I take a step back hoping to provide further context and perspective of the results – Why should I CARE ABOUT the event? What do these results REALLY represent? Then last, I provide further insight as to what operators should keep in mind when evaluating vendor offerings – What ELSE to keep in mind beyond the results in the report?

## The Big Picture

With one of the largest vendor participation ever (21 in total), this event had the potential to become very meaningful for network operators. And in my view, the event met such expectations.

The following list summarizes SR related facts pertinent to Cisco’s participation at the showcase: 
* Cisco was one of a total of ten (10) network and test equipment vendors that validated readiness of their SR implementations. The interop counted with participation from all major networking vendors
* By far, the use of SR-MPLS dominated on those test cases that relied on MPLS as a transport.  This included the transport of services such as IP VPN and Ethernet VPNs. Use of LDP was kept to a minimum. RSVP-TE was not used at the event
* IS-IS was chosen as the main IGP throughout the event. Note that use of OSPF was considered but not prioritized due to time constraints
* Baseline IS-IS SR functionality was successfully verified. No interoperability issues were observed among Cisco and vendors that we interconnected with. Verified functionality included:
  * IPv4 control plane
  * Prefix Segment ID (Prefix-SID) for host prefixes including both Node and Anycast SIDs
  * Adjacency Segment IDs (Adj-SIDs) for IS-IS adjacencies
  * Prefix-to-SID mapping advertisements performed by the SR Mapping Server (SRMS) function
* SR Traffic Engineering (SRTE) was another area of focus with validation of the following:
  * Path Computation Element Protocol (PCEP) - Stateful PCE model
  * PCEP extensions for Segment Routing
  * BGP Link-State (BGP-LS) and extensions for Segment Routing
* In addition, the following SR-MPLS related topics were tested for the first time at EANTC:
  * Topology Independent LFA (TI-LFA)
  * SR Prefix SID extensions for BGP (BGP-SR)
  * SR Operations, Administration and Maintenance (OAM)
* Lastly, SRv6 was validated also for the first time at an EANTC event. Tests covered baseline functions from the [SRv6 Network Programming](https://datatracker.ietf.org/doc/draft-filsfils-spring-srv6-network-programming/) IETF draft

I describe EVPN related facts later in the blog

