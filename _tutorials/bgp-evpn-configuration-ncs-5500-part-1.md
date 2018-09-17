---
published: true
date: '2018-09-04 10:45 -0700'
title: BGP-EVPN Configuration on NCS 5500 part-1
author: Ahmad Bilal Siddiqui
position: top
excerpt: >-
  This is the first part of the series for EVPN configuration on NCS5500. In
  this post we will focus on configuring BGP-EVPN Control-Plane & Segment
  Routing Forwarding-Plane.
tags:
  - iosxr
  - EVPN
  - NCS 5500
  - ncs5500
  - evpn
---

{% include toc %}

# Topic: Configure BGP-EVPN Control-Plane & Segment Routing based MPLS Forwarding-Plane

## Introduction to BGP-EVPN

EVPN is the next generation L2VPN technology, it provides layer-2 as well as layer-3 VPN services in a scalable and simplified manner. The evolution of EVPN started due to the need of a scalable solution to bridge various layer-2 domains and overcome the limitations faced by VPLS such as scalability, multi-homing and per-flow load balancing. 

EVPN uses MAC addresses as routable addresses and distribute them to all participating PEs via MP-BGP EVPN control-plane. EVPN is used for E-LAN, E-LINE, E-TREE services and provides data-plane and control-plane separation. This allows the use of different encapsulation mechanisms in data plane while maintaining the same control-plane. In addition, EVPN offers many advantages over existing technologies, including more efficient load-balancing of VPN traffic. Some of the prominent advantages are:
- Multi-homing and redundancy
- Per flow-based load balancing
- Scalability 
- Provisioning simplicity 
- Reduced operational complexity

In this and next few posts we will cover BGP-EVPN configuration, implementation and verification on NCS 5500 Platform using IOS-XR. The goal of this tutorial is to provide familiarity to BGP-EVPN from configuration perspective and cover the following use cases.

- [Configuring BGP EVPN control-plane and Segment Routing based forwarding plane](https://xrdocs.io/cloud-scale-networking/tutorials/bgp-evpn-configuration-ncs-5500-part-1/)
- [Configure EVPN based Multi-homing to the Hosts](https://xrdocs.io/cloud-scale-networking/tutorials/bgp-evpn-configuration-ncs-5500-part-2/)
- [EVPN based Layer-2 VPN Service](https://xrdocs.io/cloud-scale-networking/tutorials/bgp-evpn-configuration-ncs-5500-part-3/)
- EVPN-IRB between Leafs in the network



## BGP-EVPN Key Route Types for Reference

The EVPN network layer reachability information (NLRI) provides different route types. Following is the summary of the route types and their usage.

| **Route Type**  | **Usage** |
| 0x1 Ethernet Auto-Discovery (A-D) Route | MAC Mass-Withdraw, Aliasing (load balancing)|
| 0x2 MAC Advertisement Route | Advertises Host MAC and IP address |
| 0x3 Inclusive Multicast Route | Indicates interest of BUM traffic for attached L2 segments |
| 0x4 Ethernet Segment Route | Auto discovery of Multi-homed Ethernet Segments and Designated Forwarder (DF) Election |
| 0x5 IP Prefix Route | Advertises IP prefix for a subnet via EVPN address family |


**Note:** _We are using Spine Leaf Fabric example in the configuration but essentially a Leaf is a PE and Spine is a P router as we are implementing MPLS forwarding plane with BGP-EVPN._ 



## Configuring BGP EVPN control-plane and ISIS Segment Routing forwarding plane

In this post, we will configure the BGP EVPN control-plane and ISIS Segment Routing based forwarding plane. This will provide the basis to enable us for provisioning of EVPN based services using segment routing transport.


## Reference Topology:
![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/evpn-config/Reference-Topology.png?raw=true)

### Task 1: Configure the Routing Protocol for Transport:

Configure IGP routing protocol between Leafs and Spines. In this tutorial we are using ISIS as the underlay routing protocol. 

| **Loopback 0** | **Prefix-SID** | **ISIS Net** |
| Spine-1 6.6.6.6/32 | 16006 | 49.0001.0000.0000.0006.0 |
| Spine-2 7.7.7.7/32 | 16007 | 49.0001.0000.0000.0007.0 |
| Leaf-1  1.1.1.1/32 | 16001 | 49.0001.0000.0000.0001.0 |
| Leaf-2  2.2.2.2/32 | 16002 | 49.0001.0000.0000.0002.0 |
| Leaf-5  5.5.5.5/32 | 16005 | 49.0001.0000.0000.0005.0 |

Following is a sample config from Leaf-1 to implement ISIS routing protocol in the network. Similar configs with relevant Net address (shown in above table) and interfaces should be used on other devices to bring up the ISIS routing protocol in the network. Don’t configure ISIS on the links from host to leafs, these will be set up later as layer-2 links.
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    router isis 1
     is-type level-2-only
     net <mark>49.0001.0000.0000.0001.00</mark>
     nsr
     log adjacency changes
     address-family ipv4 unicast
      metric-style wide
    !
     interface Bundle-Ether16
      point-to-point
      address-family ipv4 unicast
    !
     interface Bundle-Ether17
      point-to-point
      address-family ipv4 unicast
    !        
     interface Loopback0
      passive
      address-family ipv4 unicast
    !
</code>
</pre>
</div>

Verify that the point-to-point interfaces between the spines and leafs and other devices in the network are up and the ISIS routing adjacency is formed between the devices as per the topology. In this setup, ISIS routing protocol is configured on all the devices except the hosts, the host will be connected layer-2 dual-homed to the Leafs.

The “**show isis neighbor**” and “**show route isis**” commands can be used to verify that the adjacency is formed and the routes of all the Leafs and Spines are learnt via ISIS.


### Task 2: Enable ISIS Segment Routing:

Configure Segment Routing protocol under ISIS routing protocol which enables MPLS on all the non-passive ISIS interfaces. A prefix SID is associated with an IP prefix and is manually configured from the segment routing global block (SRGB) range of labels. It is configured under the loopback interface with the loopback address of the node as the prefix. The prefix SID is globally unique within the segment routing domain.

The Prefix-SID can be an absolute value or an indexed value. In this guide, we are configuring Prefix-SID as absolute value. ISIS Segment Routing is configured in the Fabric between Leafs and Spines. 

Following is a sample config to enable Segment Routing in the network. Similar config with prefix-SID that is unique for each device in the network, should be configured on other devices (as per the above diagram) to enable ISIS Segment Routing. In this config prefix-SID is enabled on the “loopback 0” interface of the devices.

![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/evpn-config/ISIS-SR-Forwarding-Plane.png?raw=true)
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Spine-1:

    router isis 1
    address-family ipv4 unicast
      <mark>segment-routing mpls</mark>
     !
     interface Loopback0
      passive
      address-family ipv4 unicast
       <mark>prefix-sid absolute 16006</mark>
    !

    Spine-2:

    router isis 1
    address-family ipv4 unicast
      segment-routing mpls
     !
     interface Loopback0
      passive
      address-family ipv4 unicast
       <mark>prefix-sid absolute 16007</mark>
    !
</code>
</pre>
</div>

Verify that all devices that have ISIS Segment Routing configured have advertised their prefix-SIDs. Also verify the prefix-SIDs are learnt and programmed in the forwarding plane on each device. 
This output is collected from Spines; we can see that the prefix-SID labels (identified by “Pfx”) of all the Leafs and other routers are learnt and programmed in the forwarding plane along with their outgoing interfaces.
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Spine-1:
    
    RP/0/RP0/CPU0:Spine-1#show isis segment-routing label table
    Tue Sep  4 23:35:11.115 UTC

    IS-IS 1 IS Label Table
    Label         Prefix/Interface
    ----------    ----------------
    16001         1.1.1.1/32
    16002         2.2.2.2/32
    16005         5.5.5.5/32
    16006         Loopback0
    16007         7.7.7.7/32
    RP/0/RP0/CPU0:Spine-1#
    

    RP/0/RP0/CPU0:Spine-1#show mpls forwarding 
    Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
    Label  Label       or ID              Interface                    Switched    
    ------ ----------- ------------------ ------------ --------------- ------------
    16001  Pop         <mark>SR Pfx (idx 1)</mark>     BE16         192.1.6.2       0           
    16002  Pop         SR Pfx (idx 2)     BE26         192.2.6.2       0           
    16005  Pop         SR Pfx (idx 5)     BE56         192.5.6.2       0           
    16007  16007       SR Pfx (idx 7)     BE16         192.1.6.2       0           
           16007       SR Pfx (idx 7)     BE26         192.2.6.2       0           
           16007       SR Pfx (idx 7)     BE56         192.5.6.2       0           
    64000  Pop         SR Adj (idx 1)     BE16         192.1.6.2       0           
    64001  Pop         SR Adj (idx 3)     BE16         192.1.6.2       0           
    64002  Pop         SR Adj (idx 1)     BE26         192.2.6.2       0           
    64003  Pop         SR Adj (idx 3)     BE26         192.2.6.2       0           
    64004  Pop         SR Adj (idx 1)     BE56         192.5.6.2       0           
    64005  Pop         SR Adj (idx 3)     BE56         192.5.6.2       0


    Spine-2:

    RP/0/RP0/CPU0:Spine-2#show isis segment-routing label table
    Tue Sep  4 23:45:48.834 UTC

    IS-IS 1 IS Label Table
    Label         Prefix/Interface
    ----------    ----------------
    16001         1.1.1.1/32
    16002         2.2.2.2/32
    16005         5.5.5.5/32
    16006         6.6.6.6/32
    16007         Loopback0
    RP/0/RP0/CPU0:Spine-2#


    RP/0/RP0/CPU0:Spine-2#show mpls forwarding
    Tue Sep  4 23:46:40.028 UTC
    Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
    Label  Label       or ID              Interface                    Switched    
    ------ ----------- ------------------ ------------ --------------- ------------
    16001  Pop         <mark>SR Pfx (idx 1)</mark>     BE17         192.1.7.2       0      
    16002  Pop         SR Pfx (idx 2)     BE27         192.2.7.2       0      
    16005  Pop         SR Pfx (idx 5)     BE57         192.5.7.2       0      
    16006  16006       SR Pfx (idx 6)     BE17         192.1.7.2       0           
           16006       SR Pfx (idx 6)     BE27         192.2.7.2       0           
           16006       SR Pfx (idx 6)     BE57         192.5.7.2       0           
    64000  Pop         SR Adj (idx 1)     BE17         192.1.7.2       0           
    64001  Pop         SR Adj (idx 3)     BE17         192.1.7.2       0           
    64002  Pop         SR Adj (idx 1)     BE27         192.2.7.2       0           
    64003  Pop         SR Adj (idx 3)     BE27         192.2.7.2       0           
    64004  Pop         SR Adj (idx 1)     BE57         192.5.7.2       0           
    64005  Pop         SR Adj (idx 3)     BE57         192.5.7.2       0           
    RP/0/RP0/CPU0:Spine-2#
</code>
</pre>
</div>

After configuring ISIS segment routing, verify that the underlay is capable of forwarding traffic using labels assigned by segment routing. 

Below output shows traceroute from Leaf-1 to Leaf-5 using the loopback address. Trace from Leaf-1 reaches Leaf-5 via Spines using label forwarding where Spine is the PHP for Leaf-5. 
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Ping from Leaf-1 to Leaf-5:

    RP/0/RP0/CPU0:Leaf-1#ping  sr-mpls 5.5.5.5/32
    Tue Sep  4 23:40:51.032 UTC

    Sending 5, 100-byte MPLS Echos to 5.5.5.5/32,
          timeout is 2 seconds, send interval is 0 msec:

    Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
      'L' - labeled output interface, 'B' - unlabeled output interface, 
      'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
      'M' - malformed request, 'm' - unsupported tlvs, 'N' - no rx label, 
      'P' - no rx intf label prot, 'p' - premature termination of LSP, 
      'R' - transit router, 'I' - unknown upstream index,
      'X' - unknown return code, 'x' - return code 0

    Type escape sequence to abort.

    !!!!!
    Success rate is 100 percent (5/5), round-trip min/avg/max = 3/5/13 ms
    RP/0/RP0/CPU0:Leaf-1#
</code>
</pre>
</div>  

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Trace from Leaf-1 to Leaf-5
    
    RP/0/RP0/CPU0:Leaf-1#trace  sr-mpls 5.5.5.5/32  
    Tue Sep  4 23:42:06.069 UTC

    Tracing MPLS Label Switched Path to 5.5.5.5/32, timeout is 2 seconds

    Codes: '!' - success, 'Q' - request not sent, '.' - timeout,
      'L' - labeled output interface, 'B' - unlabeled output interface, 
      'D' - DS Map mismatch, 'F' - no FEC mapping, 'f' - FEC mismatch,
      'M' - malformed request, 'm' - unsupported tlvs, 'N' - no rx label, 
      'P' - no rx intf label prot, 'p' - premature termination of LSP, 
      'R' - transit router, 'I' - unknown upstream index,
      'X' - unknown return code, 'x' - return code 0

    Type escape sequence to abort.

      0 192.1.7.2 MRU 1500 [Labels: 16005 Exp: 0]
    L 1 192.1.7.1 MRU 1500 [Labels: implicit-null Exp: 0] 121 ms
    ! 2 192.5.7.2 4 ms
    RP/0/RP0/CPU0:Leaf-1#
</code>
</pre>
</div>



### Task 3: Configure the BGP-EVPN Control-Plane

MP-BGP with its various address families is used to transport specific reachability information in the network. BGP’s L2VPN-EVPN address family is capable of transporting tenant-aware/VRF-aware IP (Layer-3) and MAC (Layer-2) reachability information in MP-BGP. BGP EVPN provides the learnt information to all the devices within the network through a common control plane. BGP EVPN next-hops are going to be reachable via segment routing paths.

In this configuration guide to configure EVPN in the Fabric, we will configure iBGP EVPN, however eBGP EVPN can also be configured and is support on NCS 5500 routers. Spines are configured as the BGP EVPN Route Reflectors. Leaf-1, Leaf-2 and Leaf-5 will all be Route Reflector clients.

![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/evpn-config/EVPN-Control-Plane.png?raw=true)
Configure Spines as RR for BGP EVPN address family.
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Spine-1:

    router bgp 65001
     bgp router-id 6.6.6.6
    !
     address-family l2vpn evpn
     !
     neighbor-group RRC
      remote-as 65001
      update-source Loopback0
      address-family l2vpn evpn
       route-reflector-client
      !
     !
     neighbor 1.1.1.1
      use neighbor-group RRC
      description BGP session to Leaf-1
     !
     neighbor 2.2.2.2
      use neighbor-group RRC
      description BGP session to Leaf-2
     !
     neighbor 5.5.5.5
      use neighbor-group RRC
      description BGP session to Leaf-5
     !


    Spine-2:

    router bgp 65001
     bgp router-id 7.7.7.7
    !
     address-family l2vpn evpn
     !
     neighbor-group RRC
      remote-as 65001
      update-source Loopback0
      address-family l2vpn evpn
       route-reflector-client
      !
     !
    neighbor 1.1.1.1
      use neighbor-group RRC
      description BGP session to Leaf-1
     !
     neighbor 2.2.2.2
      use neighbor-group RRC
      description BGP session to Leaf-2
     !
    neighbor 5.5.5.5
      use neighbor-group RRC
      description BGP session to Leaf-5
     !
    !
</code>
</pre>
</div>

Use the following configuration and apply it to configure the Leaf-1 Leaf-2 and Leaf-5 to form the BGP EVPN adjacency between Leafs and Route Reflectors. 
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1:

    router bgp 65001
     bgp router-id 1.1.1.1
    !        
     address-family l2vpn evpn
     !
     neighbor 6.6.6.6
      remote-as 65001
      description "BGP session to Spine-1"
      update-source Loopback0
      address-family l2vpn evpn
      !
     !
     neighbor 7.7.7.7
      remote-as 65001
      description "BGP session to Spine-2"
      update-source Loopback0
      address-family l2vpn evpn
      !
     !
    !


    Leaf-2:

    router bgp 65001
     bgp router-id 2.2.2.2
    !        
     address-family l2vpn evpn
     !
     neighbor 6.6.6.6
      remote-as 65001
      description "BGP session to Spine-1"
      update-source Loopback0
      address-family l2vpn evpn
      !
     !
     neighbor 7.7.7.7
      remote-as 65001
      description "BGP session to Spine-2"
      update-source Loopback0
      address-family l2vpn evpn
      !
     !
    !


    Leaf-5:

    router bgp 65001
     bgp router-id 5.5.5.5
    !        
     address-family l2vpn evpn
     !
     neighbor 6.6.6.6
      remote-as 65001
      description "BGP session to Spine-1"
      update-source Loopback0
      address-family l2vpn evpn
      !
     !
     neighbor 7.7.7.7
      remote-as 65001
      description "BGP session to Spine-2"
      update-source Loopback0
      address-family l2vpn evpn
      !
     !
    !
</code>
</pre>
</div>

Use “**show bgp l2vpn evpn summary**” cli command to verify the evpn neighborship between Route Reflectors and Leafs. Below output from the Spines show that the BGP EVPN neighborship is formed between the Leafs and the Route Reflectors and the control-plane is up. 
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Spine-1:

    RP/0/RP0/CPU0:Spine-1#show bgp l2vpn evpn summary 
    BGP router identifier 6.6.6.6, local AS number 65001 

    Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
    Speaker               1          1          1          1           1           0

    Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
    1.1.1.1           0 65001       8       8        1    0    0 00:06:02          0
    2.2.2.2           0 65001       7       7        1    0    0 00:04:53          0
    5.5.5.5           0 65001       7       7        1    0    0 00:04:14          0


    Spine-2: 

    RP/0/RP0/CPU0:Spine-2#show bgp l2vpn evpn summary 
    BGP router identifier 7.7.7.7, local AS number 65001

    Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
    Speaker               1          1          1          1           1           0

    Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
    1.1.1.1           0 65001       9      10        1    0    0 00:06:50          0
    2.2.2.2           0 65001       8       8        1    0    0 00:05:43          0
    5.5.5.5           0 65001       7       7        1    0    0 00:05:03          0
</code>
</pre>
</div>

In this post we covered the configuration and verification of BGP-EVPN control-plane and ISIS-SR based MPLS forwarding plane. In the [next post](https://xrdocs.io/cloud-scale-networking/tutorials/bgp-evpn-configuration-ncs-5500-part-2/) we will leverage the EVPN control-plane and ISIS-SR to provision BGP-EVPN based Multi-Homing of devices.
