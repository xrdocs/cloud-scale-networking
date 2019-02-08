---
published: true
date: '2018-12-03 10:23 -0800'
title: IOS XR Evolution
position: hidden
excerpt: '  IOS XR Evolution.'
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
