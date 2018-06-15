---
published: true
date: '2018-06-15 14:41 -0400'
title: Persistent Load Balancing or "Sticky ECMP"
author: Xander Thuijs
excerpt: >-
  Traditional ECMP or equal cost multipath loadbalances traffic over a number of
  available paths towards a destination. When one path fails, the traffic gets
  re-shuffled over the available number of paths.
position: hidden
---


{% include toc %}
{% include base_path %}

##Introduction

This document applies to NCS5500 and ASR9000 routers and has been verified as such.
{: .notice--info}

traditional ECMP or equal cost multipath loadbalances traffic over a number of available paths towards a destination. When one path fails, the traffic gets re-shuffled over the available number of paths.

This means that a flow that was before taking path "1", could now be taking path "3" although only path "2" failed.

This reshifting occurs because the hash of althogh the flow remains the same resulting in the same bucket, but the bucket may get reassigned to a new path.

 

To understand flows, buckets and traditional ECMP a bit better, you could reference the Loadbalancing Architecture document and consult the Cisco Live ID 2904 from Las Vegas 2017.

 

While this flow redistribution is not a problem in traditional core networks, because the end to end connectivity is preserved and the user would not experience any situation from it, in data center loadbalancing this can be a problem.
