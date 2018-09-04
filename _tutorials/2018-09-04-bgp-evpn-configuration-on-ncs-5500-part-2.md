---
published: false
date: '2018-09-04 15:21 -0700'
title: BGP-EVPN Configuration on NCS 5500 part-2
---
### Topic: BGP-EVPN based Multi-Homing of devices

In this tutorial, we will cover the BGP-EVPN based Multi-Homing of devices. This will be achieved by leveraging the EVPN control-plane and MPLS based forwarding to provide redundancy and per-flow load balancing to the dual homed device.

EVPN started due to the need of a scalable solution to bridge various layer-2 domains and overcome the limitations faced by VPLS such as scalability, multi-homing and per-flow load balancing. Ethernet VPN (EVPN) provides virtual bridge connectivity between the hosts/CE devices. CE devices are connected to the PE devices which forms the service edge of the network. A CE device can be a host, a router, or a switch. There can be multiple EVPNs in the provider network. Learning between the PE routers occurs in the control plane using BGP, unlike traditional bridging, where learning occurs in the data plane.


