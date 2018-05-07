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

But seriously and in retrospective, it is fascinating to have witnessed several technology (and even political) battles and transitions that shaped the networking industry. Many technologies and entire companies have had short and unsuccessful lives. In my opinion, during the late 2000s our industry was deeply distracted and as a result wasted brain / development cycles in fruitless endeavors; remember PBT, PBB-TE, T-MPLS, MPLS-TP, OpenFlow?

However, one technology has always remained strong – and that is MPLS. During the hype days of OpenFlow, several so-called experts questioned the future of MPLS in operator networks.  At that time, Cisco was starting the journey with Segment Routing (SR). In November of 2012, Cisco Fellow Clarence Filsfils first disclosed the SR concept to network operators. And just one year later, a new working group was officially formed at the IETF - Source Packet Routing In Networking (SPRING). And in 2015, EANTC was already conducting the first public SR interop event. Fast-forward to today and this interop provides the latest proof-point of overwhelming multi-vendor support for SR.

Another technology with a successful trajectory is Ethernet VPN (EVPN). Initially, it was positioned as the next generation solution for Layer 2 VPNs and in particular for Layer 2 Data Center Interconnection (DCI) and multipoint Ethernet LAN (E-LAN) services. From there, it quickly evolved to accommodate other usecases such as point-to-point Ethernet Line (E-LINE) and point-to-multipoint Ethernet Tree (E-Tree) services. Currently, EVPN usecases cross beyond Carrier Ethernet and are deeply entrenched into the datacenter for both intra-subnet and inter-subnet forwarding. In 2013, I was part of the first public interop test of PBB-EVPN at EANTC. In the last five years, multi-vendor support and sophistication of test scenarios have been growing steadily.

But enough of the history lesson!!! Let me move onto the important part of this blog

My goal is to provide a technical overview of Cisco’s participation at this year’s interop showcase with platforms powered by the IOS XR operating system. From a technology perspective, my focus will be on SR and EVPN. Be aware that Cisco was also represented by our Datacenter product line and the Network Services Orchestrator (NSO).

First, I strongly recommend the reader to go over [EANTC’s official public whitepaper](http://www.eantc.de/en/showcases/mpls_sdn_2018). Complementing the information on their report, I take a step back hoping to provide further context and perspective of the results – Why should I CARE ABOUT the event? What do these results REALLY represent? Then last, I provide further insight as to what operators should keep in mind when evaluating vendor offerings – What ELSE to keep in mind beyond the results in the report?

## The Big Picture

With one of the largest vendor participation ever (21 in total), this event had the potential to become very meaningful for network operators. And in my view, the event met such expectations.

The following list summarizes SR related facts pertinent to Cisco’s participation at the showcase: 
* Cisco was one of a total of ten (10) network and test equipment vendors that validated readiness of their SR implementations. The interop counted with participation from all major networking vendors
* By far, SR-MPLS dominated on those test cases using MPLS as a transport.  This included the transport of services such as IP VPN and Ethernet VPN. Use of LDP was kept to a minimum. RSVP-TE was not used at the event
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

## Cisco Participating Devices

At this year’s event, Cisco IOS XR product portfolio was represented by the **Cisco ASR 9000** and **NCS 5500** product families. For the NCS 5500, it was its debut at the showcase.
Also another first time, was the participation of **Cisco IOS XRv9000** virtual router acting as a virtual SR Path Computation Element.

Let’s delve next into the main test categories …

## Topology Independent Fast Reroute using Segment Routing

It is critical for operators to provide services with SLA guarantees and to automatically restore connectivity in the case of a network component failure. By relying on Segment Routing, Topology Independent Loop Free Alternate (TI-LFA) provides a local repair mechanism to achieve this goal. With behaviors described in an IETF draft, [TI-LFA](https://datatracker.ietf.org/doc/draft-bashandy-rtgwg-segment-routing-ti-lfa/) provides key benefits, including:
* Automatic Per-Destination protection – automatic backup paths are pre-computed by the IGP for each destination (prefix). TI-LFA prepares a data-plane switch-over to be activated upon detection of the failure of a link used to reach a given destination
* Topology Independent coverage – TI-LFA provides sub-50msec link, node and local SRLG protection for ANY topology. TI-LFA provides a loop free backup path irrespective of the topologies prior the failure and after the failure
* Optimal routing – optimal routing by enforcing a backup path that is identical to the post-convergence path
* Stateless operation – based on source routing, there is no need to create additional forwarding state in the network in order to enforce a backup path

As a result of these benefits and based on our deployment experience, TI-LFA remains one of the main drivers behind SR deployments to-date. TI-LFA has been one of the key areas of execution for Cisco since we started shipping it in 2014.
With this in mind, we welcomed the addition, for the first-time, of TI-LFA testcases to EANTC’s interop. Highlights of Cisco’s participation include:
* Cisco successfully validated sub-50 msec protection with TI-LFA
* Cisco successfully validated TI-LFA with Link protection
* Cisco successfully validated TI-LFA with Local SRLG protection. Cisco was the only participating vendor that passed this test case
* Note that TI-LFA with Node protection is also supported by Cisco’s implementation but was not part of the test plan. Stay tuned for upcoming announcements of new enhancements to Cisco’s TI-LFA in the 2018 summer timeframe!!!

Lastly, here are key aspects NOT COVERED by the report and that MUST always be considered as you evaluate TI-LFA implementations:
* Does the vendor implementation provide a backup path computed for each destination? Watch for implementations that may cut corners and not compute an optimum backup path for each destination in the network. Cisco’s TI-LFA implementation was designed to meet this goal
* Does the vendor implementation provide prefix-independent convergence? Make sure to validate that the implementation’s performance during activation of backup paths does NOT degrade as the number of protected prefixes increases. Cisco’s TI-LFA implementation was also designed and implemented with this principle in mind 
* Does the vendor implementation provide protection to traffic that originally is forwarded using other paradigms such as LDP signaling or pure IP-routed traffic? Make sure to validate that the implementation’s coverage includes also non-SR traffic. Cisco’s TI-LFA can also be used to protect LDP and IP traffic

For more information, I suggest reviewing this [TI-LFA tutorial](http://www.segment-routing.net/tutorials/2016-09-27-topology-independent-lfa-ti-lfa/) and [TI-LFA demonstration](http://www.segment-routing.net/demos/2016-demo-topology-independent-lfa/)



