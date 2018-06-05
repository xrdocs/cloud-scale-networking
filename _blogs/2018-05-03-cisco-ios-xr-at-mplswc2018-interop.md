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
position: top
excerpt: >-
  Learn the technical details regarding Cisco’s participation at MPLS WC 2018
  interop showcase with platforms powered by the IOS XR operating system. This
  blog will focus on Segment Routing and Ethernet VPN test areas.
---

{% include toc icon="table" title="Cisco IOS XR at MPLS WC 2018 interop" %}
{% include base_path %}

A **Chinese** translation of this blog can be found [HERE](https://www.cisco.com/c/zh_cn/solutions/service-provider/segment_routing/index.html?dtid=osowbo000775) 

## Time for another interop

I have represented Cisco for more than 8 years at EANTC-sponsored interoperability events. Going back to 2008, I have experienced more pleasant Berlin winters than I could have ever imagined.

But seriously and in retrospective, it is fascinating to have witnessed several technology (and even political) battles and transitions that shaped the networking industry. Many technologies and entire companies have had short and unsuccessful lives. In my opinion, during the late 2000s our industry was deeply distracted and as a result wasted brain / development cycles in fruitless endeavors; remember PBT, PBB-TE, T-MPLS, MPLS-TP, OpenFlow?

However, one technology has always remained strong – and that is MPLS. During the hype days of OpenFlow, several so-called experts questioned the future of MPLS in operator networks.  At that time, Cisco was starting the journey with Segment Routing (SR). In November of 2012, Cisco Fellow Clarence Filsfils first disclosed the SR concept to network operators. And just one year later, a new working group was officially formed at the IETF - Source Packet Routing In Networking (SPRING). And in 2015, EANTC was already conducting the first public SR interop event. Fast-forward to today and this interop provides the latest proof-point of overwhelming multi-vendor support for SR.

Another technology with a successful trajectory is Ethernet VPN (EVPN). Initially, it was positioned as the next generation solution for Layer 2 VPNs and in particular for Layer 2 Data Center Interconnection (DCI) and multipoint Ethernet LAN (E-LAN) services. From there, it quickly evolved to accommodate other usecases such as point-to-point Ethernet Line (E-LINE) and point-to-multipoint Ethernet Tree (E-Tree) services. Currently, EVPN usecases cross beyond Carrier Ethernet and are deeply entrenched into the datacenter for both intra-subnet and inter-subnet forwarding. In 2013, I was part of the first public interop test of PBB-EVPN at EANTC. In the last five years, multi-vendor support and sophistication of test scenarios have been growing steadily.

But enough of the history lesson!!! Let me move onto the important part of this blog

My goal is to provide a technical overview of Cisco’s participation at this year’s interop showcase with platforms powered by the IOS XR operating system. From a technology perspective, my focus will be on SR and EVPN test areas. Be aware that Cisco was also represented by our Datacenter product line and the Network Services Orchestrator (NSO).

First, I strongly recommend the reader to go over [EANTC’s official public whitepaper](http://www.eantc.de/en/showcases/mpls_sdn_2018). Complementing the information on their report, I take a step back hoping to provide further context and perspective of the results – **_Why should I CARE ABOUT the event? What do these results REALLY represent?_** Then last, I provide further insight as to what operators should keep in mind when evaluating vendor offerings – **_What ELSE to keep in mind beyond the results in the report?_**

## The Big Picture

With one of the largest vendor participation ever (21 in total) and over 60 device types, this event had the potential to become very meaningful for network operators. And in my view, the event met such expectations.

![IMG_20180412_115001.jpg]({{site.baseurl}}/images/IMG_20180412_115001.jpg)
Interop booth at MPLS WC - &copy; Photo by EANTC

The following list summarizes **SR related facts pertinent to Cisco’s participation** at the showcase: 
* **Cisco was one of a total of ten (10) network and test equipment vendors** that validated readiness of their SR implementations. The interop counted with participation from all major networking vendors
* By far, **SR-MPLS dominated on those test cases using MPLS as a transport**.  This included the transport of services such as IP VPN and Ethernet VPN. Use of LDP was kept to a minimum. RSVP-TE was not used at the event
* IS-IS was chosen as the main IGP throughout the event. Note that use of OSPF was considered but not prioritized due to time constraints
* Baseline **IS-IS SR extensions were successfully verified**. No interoperability issues were observed among Cisco and vendors that we interconnected with. Verified functionality included:
  * IPv4 control plane
  * Prefix Segment ID (Prefix-SID) for host prefixes including both Node and Anycast SIDs
  * Adjacency Segment IDs (Adj-SIDs) for IS-IS adjacencies
  * Prefix-to-SID mapping advertisements performed by the SR Mapping Server (SRMS) function
* **SR Traffic Engineering (SRTE) was another area of focus** with validation of the following:
  * Path Computation Element Protocol (PCEP) - Stateful PCE model
  * PCEP extensions for Segment Routing
  * BGP Link-State (BGP-LS) and extensions for Segment Routing
* In addition, the following SR-MPLS related topics were **tested for the first time at EANTC**:
  * **Topology Independent LFA (TI-LFA)**
  * **SR Prefix SID extensions for BGP (BGP-SR)**
  * **SR Operations, Administration and Maintenance (OAM)**
* Lastly, **SRv6** was validated also for the first time at an EANTC event. Tests covered baseline functions from the [SRv6 Network Programming](https://datatracker.ietf.org/doc/draft-filsfils-spring-srv6-network-programming/) IETF draft

I describe EVPN related facts later in the blog

## Cisco Participating Devices

At this year’s event, Cisco IOS XR product portfolio was represented by the **Cisco ASR 9000** and **NCS 5500** product families. For the NCS 5500, it was its debut at the showcase.
Also another first time, was the participation of **Cisco IOS XRv9000** virtual router acting as a virtual SR Path Computation Element.

Let’s delve next into the main test categories …

## Topology Independent Fast Reroute using Segment Routing

It is critical for operators to provide services with SLA guarantees and to automatically restore connectivity in the case of a network component failure. By relying on Segment Routing, Topology Independent Loop Free Alternate (TI-LFA) provides a local repair mechanism to achieve this goal. With behaviors described in an IETF draft, [TI-LFA](https://datatracker.ietf.org/doc/draft-bashandy-rtgwg-segment-routing-ti-lfa/) provides key benefits, including:
* **Automatic Per-Destination protection** – automatic backup paths are pre-computed by the IGP for each destination (prefix). TI-LFA prepares a data-plane switch-over to be activated upon detection of the failure of a link used to reach a given destination
* **Topology Independent coverage** – TI-LFA provides sub-50msec link, node and local SRLG protection for ANY topology. TI-LFA provides a loop free backup path irrespective of the topologies prior the failure and after the failure
* **Optimal routing** – TI-LFA provides optimal routing by enforcing a backup path that is identical to the post-convergence path
* **Stateless operation** – based on source routing paradigm, there is no need to create additional forwarding state in the network in order to enforce a backup path

As a result of these benefits and based on our deployment experience, TI-LFA remains one of the main drivers behind SR deployments to-date. **TI-LFA has been one of the key areas of execution for Cisco since we started shipping it in 2014**.

With this in mind, we welcomed the addition, for the first-time, of TI-LFA testcases to EANTC’s interop. Highlights of Cisco’s participation include:
* **Cisco successfully validated sub-50 msec protection with TI-LFA**
* **Cisco successfully validated TI-LFA with Link protection**
* **Cisco successfully validated TI-LFA with Local SRLG protection. Cisco was the only participating vendor that passed this test case**
* Note that TI-LFA with Node protection is also supported by Cisco’s implementation but was not part of the test plan. Stay tuned for upcoming announcements of new enhancements to Cisco’s TI-LFA in the 2018 summer timeframe!!!

Lastly, here are key aspects **NOT COVERED** by the report and that MUST always be considered as you evaluate TI-LFA implementations:
{: .notice--warning}
* Does the vendor implementation provide a backup path computed for each destination? Watch for implementations that may cut corners and not compute an optimum backup path for each destination in the network. Cisco’s TI-LFA implementation was designed to meet this goal
* Does the vendor implementation provide prefix-independent convergence? Make sure to validate that the implementation’s performance during activation of backup paths does NOT degrade as the number of protected prefixes increases. Cisco’s TI-LFA implementation was also designed and implemented with this principle in mind 
* Does the vendor implementation provide protection to traffic that originally is forwarded using other paradigms such as LDP signaling or pure IP-routed traffic? Make sure to validate that the implementation’s coverage includes also non-SR traffic. Cisco’s TI-LFA can also be used to protect LDP and IP traffic
{: .notice--warning}

For more information, I suggest reviewing this [TI-LFA tutorial](http://www.segment-routing.net/tutorials/2016-09-27-topology-independent-lfa-ti-lfa/) and [TI-LFA demonstration](http://www.segment-routing.net/demos/2016-demo-topology-independent-lfa/)

## SR Traffic Engineering (SRTE) and Path Computation Element Protocol (PCEP)

Segment Routing’s ability to address traffic engineering (TE) use cases is one of the most sought-after applications of the technology.

The [SRTE architecture IETF draft](https://datatracker.ietf.org/doc/draft-filsfils-spring-segment-routing-policy/) describes in detail the behaviors and mechanisms that allow a headend node to direct traffic along a path in the network using “SR Policies”. Based on the source routing paradigm, all state is encoded at the headend node using an ordered list of segments. As a result, SRTE no longer requires state to be maintained at intermediate nodes as it was the case with legacy TE solutions.

The segment routed path of an SR policy can be derived from a number of choices, including computation by a centralized PCE. An IETF draft describes [PCEP extensions](https://datatracker.ietf.org/doc/draft-ietf-pce-segment-routing/) for SR that allow a stateful PCE to compute and initiate TE paths, as well as a path computation client (PCC) to request a path subject to certain constraint(s) and optimization criteria in SR networks.

Overall, this was one of the MOST active areas at the interop and where I personally spent most time on. With one of the largest number of positive results achieved, EANTC reported over 30 successful combinations of different PCE-PCC vendor / products.

Cisco’s participation in this test area can be broken into 2 categories – as a PCC and as a PCE.

Highlights of Cisco’s participation as PCC include:
* **Cisco was one of a group of six (6) vendors (not counting traffic emulator vendors) that participated as PCEP PCC headend nodes**
* **Cisco’s SR PCC implementation was the MOST interoperable PCC at the event considering the number of successful test results against participating PCE vendors**
* **As PCC, Cisco successfully validated the creation, update and deletion of SR policies based on the PCE-initiated model with all participating non-Cisco SR PCEs**
* **As PCC, Cisco successfully validated the creation, update and deletion of SR policies based on the PCC-initiated model with all participating non-Cisco SR PCEs**

Highlights of Cisco’s participation as PCE include:
* **Cisco was one of a group of three (3) vendors (not counting traffic emulator vendors) that participated as PCEP PCE nodes**
* **Cisco’s SR PCE was the MOST interoperable PCE at the event considering the number of successful test results across participating PCC vendors**
* **As PCE, Cisco successfully validated the creation, update and deletion of SR policies based on the PCE-initiated model with participating non-Cisco SR PCCs**
* **As PCE, Cisco successfully validated the creation, update and deletion of SR policies using PCC-initiated model with participating non-Cisco SR PCCs**
* **Also, Cisco SR PCE successfully validated single-domain and multi-domain topology learning using BGP-LS feeds originated at non-Cisco nodes**
* **Lastly, Cisco SR PCE was the only PCE at the event to successfully validate path computation on a multi-domain network with Egress Peering Engineering (EPE) SIDs at domain boundaries**

![20180411_163350.jpg]({{site.baseurl}}/images/20180411_163350.jpg)
Presenting at MPLS WC with colleagues from Huawei (left) and Nokia (right) - &copy; Photo by EANTC


Lastly, and beyond protocol interoperability, it is important that operators consider these key aspects **NOT COVERED** by the report when evaluating SRTE headend and PCE implementations:
{: .notice--warning}
* Does the PCE implementation provide path computation based on the SR principles – i.e. maximizing ECMP and minimizing label stack size? Watch for implementations that again may cut corners and simply reuse RSVP-TE algorithms for SR. RSVP-TE is non-ECMP aware and circuit-based and hence requiring many SIDs when coding an SR path. Cisco developed NEW algorithms for SR path computation with [recognized innovation by the academic community](http://conferences.sigcomm.org/sigcomm/2015/pdf/papers/p15.pdf)
* Does the vendor implementation allow for path computation at the headend? For the majority of single-domain scenarios, the headend node should be in a position to compute paths. Therefore an operator should not settled for SR-optimized path computation only at the PCE but also at the PCC. Remember that the main difference between a headend node and a PCE node is the scope/size of the topology database. The former is single domain while the later could be multi-domain. A Cisco SRTE headend is provided with the SAME computation algorithms that are present in the PCE
* Does the implementation provide maximum scale without requiring a-priori full-mesh connectivity? Triggered by a service route (e.g. IP VPN), Cisco’s SR On-demand Next Hop (SR ODN) provides a LOCAL mechanism at the headend that triggers instantiation of an SR policy enforcing the transport SLA required by the service. Learn more and [watch an ODN demonstration here](http://www.segment-routing.net/demos/2016-demo-on-demand-next-hop-large-scale-design/)
* Does the implementation avoid complex and many times performance-impacting traffic steering techniques? Based on the [steering behaviors](https://tools.ietf.org/html/draft-filsfils-spring-segment-routing-policy-05#section-8.4) described in the SRTE architecture IETF draft, Cisco supports an innovative steering technique that we called Automated Steering (AS). AS steers automatically service traffic onto the right SR policy based on the color of the service route. This solution provides simplicity and performance without penalties. The benefits of AS apply to all instantiation methods of an SR policy (e.g. on-demand, pce-initiated, local). Watch a demonstration of AS in the same ODN demonstration highlighted in the previous bullet
{: .notice--warning}

For more information, I recommend reviewing this [SRTE tutorial](http://www.segment-routing.net/tutorials/2017-03-06-segment-routing-traffic-engineering-srte/)

## SR and LDP Interworking

One of the key usecases addressed by SR is the support of brownfield deployments. [Segment Routing interworking with LDP](https://datatracker.ietf.org/doc/draft-ietf-spring-segment-routing-ldp-interop/) IETF draft documents several mechanisms through which SR interworks with LDP in a network where a mix of SR and LDP routers co-exist.

For scenarios where SR and LDP are available in different parts of the network, a continuous MPLS LSP in the SR-to-LDP direction leverages the so-called **SR Mapping Server (SRMS)** function. The SRMS is an IGP node advertising mapping between Segment Identifiers (SID) and prefixes advertised by other IGP nodes.

**Cisco’s SR implementation supports SRMS and SR/LDP data-plane interworking functions since 2014**.

Highlights of Cisco’s participation on this test case include:
* **Cisco was successfully validated as an SR-only node receiving IS-IS SRMS advertisements from a non-Cisco SRMS implementation**
* **Cisco was successfully validated as an SRMS node in a domain with non-Cisco SR-only nodes**
* **Cisco was successfully validated as an LDP/SR interconnect “stitching” node**

For more information, I suggest reviewing this [SRMS tutorial](http://www.segment-routing.net/tutorials/2016-09-27-segment-routing-mapping-server/) and [SR / LDP interworking tutorial](http://www.segment-routing.net/tutorials/2016-09-27-segment-routing-and-ldp-interworking/)


## SR Prefix SID extensions for BGP (BGP-SR)

[Segment Routing Prefix SID extensions for BGP](https://datatracker.ietf.org/doc/draft-ietf-idr-bgp-prefix-sid/) IETF draft defines a BGP attribute for announcing BGP Prefix Segment Identifiers (BGP Prefix-SID) information.  A BGP Prefix-SID is always a global segment (a global instruction) and it identifies an instruction to forward the packet over the ECMP-aware best-path computed by BGP to the related prefix.

Use cases for the BGP Prefix SID are documented in these IETF drafts: [BGP-Prefix Segment in large-scale data centers](https://datatracker.ietf.org/doc/draft-ietf-spring-segment-routing-msdc/) and [Interconnecting Millions Of Endpoints With Segment Routing](https://datatracker.ietf.org/doc/draft-filsfils-spring-large-scale-interconnect/).

This test case represented a first-time interop test at EANTC. Highlights of Cisco’s participation on this test case include:
* **Cisco was successfully validated as a Leaf node in a multi-vendor BGP-SR fabric**
* **Cisco was successfully validated as a Spine node in a multi-vendor BGP-SR fabric**

## SR Operations, Administration and Maintenance (OAM)

Network operators require the ability to verify and isolate faults within the SR network. IETF [RFC 8287](https://tools.ietf.org/html/rfc8287) defines a set of extensions to perform LSP Ping and Traceroute operations for SR IGP-Prefix SIDs and IGP-Adjacency SIDs with an MPLS data plane.

This test case also represented a first-time interop test at EANTC. Highlights of Cisco’s participation on this test case include:
* **Cisco was successfully validated as initiator of SR OAM ping / traceroute operations  - using an MPLS echo request with a target FEC Stack TLV carrying FECs with the new IPv4 IGP-prefix SID sub-TLV**
* **Cisco was successfully validated as target / responder of SR OAM ping / traceroute operations**

During the event, an interop issue arose among some vendors due to different interpretations of RFC 8287 concerning the IPv4 IGP-prefix SID sub-TLV length. A [technical errata](https://www.rfc-editor.org/errata_search.php?rfc=8287) was raised by one of the interop participating vendors.

## Ethernet VPN

From its inception, Cisco has been leading the definition of EVPN at the IETF. And followed by a strong commitment reflected in our implementation across Service Provider and Datacenter product lines, the technology is deployed by network operators worldwide.

Though some may have noticed IOS XR’s absence for the past couple of years at this interop, we returned back with full-strength and showcased the advanced EVPN feature set available in Cisco ASR 9000 and NCS 5500 product families.

Highlights of Cisco’s participation on this test area include:
* **Cisco was one of a group of eight (8) vendors (not counting traffic emulator vendors) that participated in the EVPN test area**. This represents an all-time high and included participation from all major networking vendors
* **Cisco acted as the main BGP route-reflector for EVPN and was leveraged by all participating vendors connected to the SR-MPLS core** 
* For the first time at EANTC, **a common SR-MPLS network was used as the main transport for EVPN services across the core**
* **EVPN all-active multi-homing over SR-MPLS test case**
  * All-active multi-homing functionality is one of the main advantages of EVPN over its legacy predecessors such as VPLS
  * Cisco was successfully validated as a PE in a multi-vendor multi-home Ethernet segment
* **EVPN VPWS over SR-MPLS test case**
  * Cisco was successfully validated as a PE in a single-home configuration. Participating vendors had agreed to perform multi-home testing, but we ran out of time. _There are only so many days that one can run non-stop on high caffeine!!!_
* **EVPN-VXLAN and IP-VPN Interworking test case**
  * Cisco was successfully validated as a Layer 3 DCI interconnecting EVPN datacenter sites across a WAN network based on IP-VPN 

## What is NEXT?
Follow us on [Twitter](https://twitter.com/segmentrouting) and [LinkedIN](https://www.linkedin.com/groups/8266623) for the latest announcements

Also visit our [external SR site](http://www.segment-routing.net/) to stay abreast of the latest presentations, tutorials, demonstrations and much more!!!

Interop-wise, I look forward to another successful event in 2019. In particular, I look forward to multi-vendor interest and readiness in a number of important standard-based solutions that Cisco ALREADY proposed for this year’s event; including:
* **IGP Segment Routing Flexible Algorithms** – Flex Algo, the latest addition to the SRTE toolkit, defines IGP extensions that allow an operator to customize and assign traffic-engineering optimizations to an IGP prefix algorithm. [Flex Algo](https://datatracker.ietf.org/doc/draft-ietf-lsr-flex-algo/) is defined at IETF for both IS-IS and OSPF. Cisco announced this new solution at Cisco Live Barcelona 2018 and further described it in [Clarence’s blog](https://blogs.cisco.com/sp/flexible-algorithm-makes-segment-routing-traffic-engineering-even-more-agile) and this [demonstration](http://www.segment-routing.net/demos/2018-sr-igp-flex-alg-demo/)
* **BGP-signaled Segment Routing Policies** – BGP can also be used to signal an SR Policy candidate path to an SRTE headend.  A new BGP SAFI and NLRI are [under standardization at IETF](https://datatracker.ietf.org/doc/draft-ietf-idr-segment-routing-te-policy/)
* **PCEP LSP Disjointness Signaling and Computation** – Path diversity is a very common use case today in IP/MPLS networks especially for Layer 2 transport over MPLS. An [IETF draft](https://datatracker.ietf.org/doc/draft-ietf-pce-association-diversity/) describes a PCEP extension for signaling LSP diversity constraint. Such request can then be honored by a PCE computing LSP paths of the desired disjointness type

But above all, I really look forward to yet another pleasant Berlin winter!!!
