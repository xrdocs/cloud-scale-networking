---
published: false
date: '2017-08-07 13:46 +0200'
title: Understanding NCS5500 Resources (S01E03)
author: Nicolas Fevrier
excerpt: Third post on the NCS5500 Resources focusing on IPv6 prefixes
position: hidden
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %} 

## S01E03 IPv6 Prefixes

### Previously on "Understanding NCS5500 Resources"

In the previous posts, we introduced the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/) and we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/).

All the principles described below and the examples used to illustrate them were validated in August 2017 with Jericho-based systems, using scale (with eTCAM) and base (without eTCAM) line cards and running the two IOS XR releases available: 6.1.4 and 6.2.2.
{: .notice--info}

### IPv6 routes and FIB Profiles

Spend a few minutes to read the S01E02 to understand the different databases used to store routes in NCS5500:

![Resources]({{site.baseurl}}/images/resources.jpg){: .align-center}

- **LPM**: Longest Prefix Match Database (or KAPS) is an SRAM used to store IPv4 and IPv6 prefixes. 
- **LEM**: Large Exact Match Database also used to store specific IPv4 and IPv6 routes, plus MAC addresses and MPLS labels.
- **eTCAM**: external TCAMs, only present in the -SE "scale" line cards and systems. As the name implies, they are not a resource inside the Forwarding ASIC, it's an additional memory used to extend unicast route and ACL / classifiers scale.

In the same former post, we explained how the different profiles influenced the prefixes storing in different databases for base and scale systems or line cards.

Thinks will be simpler with IPv6 prefixes since the order of operation will be exactly the same, regardless of the FIB profile activated and regardless of the type of line card (base or scale):

![IPv6-order.jpg]({{site.baseurl}}/images/IPv6-order.jpg){: .align-center}

The logic behind this decision is that /48 prefixes are by far the largest population of the public table:

![ipv6-table.jpg]({{site.baseurl}}/images/ipv6-table.jpg){: .align-center}
(From https://twitter.com/bgp6_table){: .align-center}

The IPv6 route distribution will be the following.

![IPv6-base-host-distr.jpg]({{site.baseurl}}/images/IPv6-base-host-distr.jpg){: .align-center}
Base systems with Host-optimized FIB profile{: .align-center}

![IPv6-base-internet-distr.jpg]({{site.baseurl}}/images/IPv6-base-internet-distr.jpg){: .align-center}
Base systems with Internet-optimized FIB profile{: .align-center}

![IPv6-scale-distribution.jpg]({{site.baseurl}}/images/IPv6-scale-distribution.jpg){: .align-center}
Scale systems regardless of FIB profile{: .align-center}

### Lab verification

![IPv6-47-0.jpg]({{site.baseurl}}/images/IPv6-47-0.jpg){: .align-center}

![IPv6-48.jpg]({{site.baseurl}}/images/IPv6-48.jpg){: .align-center}

![IPv6-128-49.jpg]({{site.baseurl}}/images/IPv6-128-49.jpg){: .align-center}






