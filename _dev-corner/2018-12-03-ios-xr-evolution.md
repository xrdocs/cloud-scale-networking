---
published: true
date: '2018-12-03 10:23 -0800'
title: IOS XR Evolution
position: hidden
excerpt: Inside IOS Xr
author: Jag Tangirala
tags:
  - iosxr
  - cisco
---

{% include toc %}
{% include base_path %}


## Introduction
This blog is the first in a series about IOS XR’s software architecture. I believe that many of today’s best-practice architecture/design patterns for a successful buildout of scalable network system, are rooted in the IOS XR's architecture.

In this series, I will explore the part of IOS XR that is typically invisible, and describe IOS XR's architecture from its foundations up. I go into why the architecture looks like it does, introduce key design patterns, and draw occasional parallels with other scalable and high-performance software.                                                                                                                                                                                               
## IOS XR Evolution

When IOS XR started its journey several years ago, Google, Facebook, AWS etc., were not around. The world had not yet begun to massively scale!

Many of the massively scalable distributed system patterns that are perhaps more familiar to software developers these days were not that popular yet (mostly limited to text books or were part of a few closed commercial software implementations).

There weren't yet messaging infrastructures like ActiveMQ, RabbitMQ, ZeroMQ, Nanomsg, Kafka etc., in-memory databases like Redis, Memcached, Hazelcast etc., or distributed file systems (DFS) like HDFS, GlusterFS, CEPH etc.

Around this time, Cisco had embarked on building the next generation Network Operating System (NOS) with one clear vision -  building a highly scalable, reliable, available, modifiable, high performance NOS for the Service Provider (SP) space that caters all the way from low end single chassis systems to massive high end multi-chassis routers. Pulling this off required leveraging the deep networking knowledge base from the Cisco IOS operating system. In addition, to meet the rigorous SP requirements for this NOS, a slew of ground breaking infrastructure and distributed systems architecture/design patterns (depicted in the following picture) were brought into the system.

![]({{site.baseurl}}/images/dev-corner/xr_ev/1_image2018-9-14_9-48-40.png){: .align-center}

Before we get into the details of the above IOS XR architecture patterns in detail in the rest of the blog, it is worth quickly going through a summary of how the IOS XR has evolved over the years. There are several dimensions across which the IOS XR has demonstrated its modularity and adaptability capabilities with some major evolutions over the years. The following table summarizes these evolutions:

![]({{site.baseurl}}/images/dev-corner/xr_ev/2_table.png){: .align-center}

_Evolution from Core to Core and Edge_


IOS XR started off its journey as a NOS for high-end core routers like CRS and moved into the edge domain with the ASR9K series. The SP carrier class router demands from core and edge domains, and differing platform variations/features/packages between these two domains, have validated and strengthened  IOS XR's modularity.


_Evolution from QNX to Linux_

IOS XR started with QNX as its base OS and then moved to the 64-bit Linux as the base OS. This transition affects the IOS XR's event management, timers, syslog, messaging, pulses, file system, resource management, drivers etc., but due to the IOS XR's modular architecture, the transition turned out to be smooth. 

_Evolution from modular to small form factor systems:_

Another evolution the IOS XR has made is its transition to support small form factor platforms (single CPU, 1RU/2RU boxes) and centralized platforms along with modular platforms where it started its journey. This reflects the IOS XR's ability to not just scale up but scale down.

_Evolution from routing to routing and optical networking products_

IOS XR has branched itself to support NCS400, NCS1000 like optical networking products. This required bringing in various optical networking specific software stacks and modularizing/removing some routing stacks/features.

_Evolution from custom silicon to merchant silicon_

IOS XR is designed to originally support high performance and feature/resource rich custom ASICs as the forwarding engines. Yet recently it has made a smooth transition to support various merchant silicon ASICs and forwarding paradigms. This is possible due to proper hardware forwarding abstractions and modularity built into the IOS XR's architecture.

_Evolution from Cisco only hardware to Cisco hardware, Whiteboxes and virtual routers_

Supporting whiteboxes or running as a virtual router requires right hardware abstractions at the lower layers. IOS XR has recently evolved to support these use cases.

_Evolution from CLI/SNMP to Telemetry/Programmable Interfaces_

CLI and SNMP are the traditional router management interfaces. But over the last few years, there is a major push towards programmable interfaces via YANG models and data streaming via Telemetry. The IOS XR has evolved to support this need by leveraging the strengths of its model driven datastore SysDB (more on this later) and extending its capabilities.
Each of the above is a significant evolution and demonstrates IOS XR's ability to evolve and take on new challenges.

Each of the above is a significant evolution and demonstrates IOS XR's ability to evolve and take on new challenges ✅
{: .notice--success}


## The IOS XR Architecture Strategy

So how does one architect a carrier grade NOS?

The Cisco IOS XR Network Operating System (NOS) is developed by not changing the solution to an existing problem but rather asking a different question by looking at the customer's networking needs for decades to come.  The high level and heavily simplified IOS XR architecture strategy can be explained via the following six steps:

![]({{site.baseurl}}/images/dev-corner/xr_ev/3_strategy.png){: .align-center}

- Appropriate higher level abstractions capture the essence of the system and they are key in driving the subsequent architecture/design patterns.
- Once abstractions are in place, IOS XR has focused on large state management in a router. (State is simply the condition or quality of an entity at an instant in time and it is usually represented by data in the system; hence, state and data are used interchangeably in this blog).
- Once the state generation and distribution patterns are modeled correctly, the next step is the design and placement of the processes that work with this state.
- But processes in a router interact with each other within a node (intra-node) and across the nodes (inter-node), often times moving significant amount of data with specific latency requirements. Hence the next important logical step in the architecture is designing a high-performance messaging infrastructure.
- The final step, once the messaging infrastructure is in place, is to understand different data access and distribution patterns of various applications over the messaging infrastructure and design those constructs.
- The high availability and upgradeability considerations span across all the stages.


In the rest of the blog, we dig into the internals of each of the IOS XR architecture strategy steps, and discuss principles and trade-offs in each step. On this journey, we will try to find useful ways of thinking about IOS XR NOS—not just how it is architected, but also why it is architected that way, and what to look for in a good NOS in general.


## Decoupled Planes Abstraction

IOS XR is a multi-process, distributed network operating system with tall order goals as mentioned earlier. In order to deliver on those goals, strong architectural abstractions are necessary. The IOS XR is architecturally divided into the following three planes:

- Management plane
- Control plane
- Data plane

These planes are a categorization of the traffic handled by a router and they provide an abstraction for the architecture of the router software. The planes abstraction helps in hiding a great deal of implementation detail behind a clean and nice facade. The management plane implements the external user interface used by operators to configure and query the system. The control plane is responsible for determining routes to use for traffic flows and generally how traffic should be forwarded. Control plane protocols (e.g. routing protocols) exchange information with other devices. Data plane directs traffic flows through the device. Forwarding is typically performed by hardware (but can be software) and data plane software is responsible for setting up hardware to perform forwarding.


![]({{site.baseurl}}/images/dev-corner/xr_ev/4_arch.png){: .align-center}

While the above three planes are perhaps more well known, there is a relatively less visible Infrastructure Plane behind the three planes, providing various key architecture/design patterns that are the subject of the rest of this blog.



![]({{site.baseurl}}/images/dev-corner/xr_ev/5_infra.png){: .align-center}