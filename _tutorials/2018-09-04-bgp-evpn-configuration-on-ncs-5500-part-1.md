---
published: true
date: '2018-09-04 10:45 -0700'
title: BGP-EVPN-configuration on NCS 5500-part-1
author: Ahmad Bilal Siddiqui
position: hidden
---

##Introduction to BGP-EVPN

EVPN is the next generation L2VPN technology, it provides layer-2 as well as layer-3 VPN services in a scalable and simplified manner. The evolution of EVPN started due to the need of a scalable solution to bridge various layer-2 domains and overcome the limitations faced by VPLS such as scalability, multi-homing and per-flow load balancing. 

EVPN uses MAC addresses as routable addresses and distribute them to all participating PEs via MP-BGP EVPN control-plane. EVPN is used for E-LAN, E-LINE, E-TREE services and provides data plane and control plane separation. Allowing use of different encapsulation mechanisms in data plane while maintaining the same control plane. EVPN offers many advantages over existing technologies, including more efficient load-balancing of VPN traffic. Some of the prominent advantages are:

	•	Multi-homing and redundancy
	•	Per flow-based load balancing
	•	Scalability 
	•	Provisioning simplicity 
	•	Reduced operational complexity

In this and next few posts we will cover BGP-EVPN configuration, implementation and verification on NCS 5500 Platform using IOS-XR. The goal of this series is to provide familiarity to BGP-EVPN from configuration perspective and cover the following use cases.

	•	Configuring BGP EVPN control-plane and ISIS Segment Routing forwarding plane
	•	EVPN based Layer-2 VPN stretch with in the Fabric
	•	EVPN-IRB between Leafs within the Fabric
	•	Configure EVPN based Multi-homing to the Hosts
	•	Provision EVPN Layer-3 stretch with EVPN and VPNv4 interworking


##BGP-EVPN Key Route Types for Reference

The EVPN network layer reachability information (NLRI) provides different route types. Following is the summary of the route types and their usage.

| Route Type  | Usage |
| 0x1 Ethernet Auto-Discovery (A-D) Route | MAC Mass-Withdraw 
Aliasing (load balancing)|
| 0x2 MAC Advertisement Route | Advertises Host MAC and IP address |
| 0x3 Inclusive Multicast Route | Indicates interest of BUM traffic for attached L2 segments |
| 0x4 Ethernet Segment Route | Auto discovery of Multi-homed Ethernet Segments 
and Designated Forwarder (DF) Election |
| 0x5 IP Prefix Route | Advertises IP prefix for a subnet via EVPN address family |


#Disclaimer

This document is to familiarize with BGP-EVPN. The lab design and configuration examples in this document can be used as a reference, however it’s not a design best practices guide. Thus, not all recommended features are used, or enabled optimally.  

**Note:** We are using Spine Leaf Fabric example in the configuration but essentially a Leaf is a PE and Spine is a P router as we are implementing MPLS forwarding plane with BGP-EVPN. 



##Configuring BGP EVPN control-plane and ISIS Segment Routing forwarding plane

In this post, we will configure the BGP EVPN control-plane and ISIS Segment Routing based forwarding plane. This will provide the basis to enable us for provisioning of EVPN based services using segment routing transport.


##Reference Topology:



##Task 1: Configure the Fabric Underlay Routing Protocol:

Configure IGP routing protocol between Leafs and Spines. In this config guide we are using ISIS as the underlay routing protocol. 

Following is a sample config from Leaf-1, to configure ISIS routing protocol in the network. Similar config with relevant Net address (shown in above table) and interfaces should be configured on other devices to bring up the ISIS routing protocol in the network. Don’t configure ISIS on the links from host to leafs, these will be configured later as layer-2 links.


    router isis 1
     is-type level-2-only
     net 49.0001.0000.0000.0001.00
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


##Task 2: Verify the Fabric Underlay Routing Protocol:

Verify that the point-to-point interfaces between the spines and leafs and other devices in the network are up and the ISIS routing adjacency is formed between the devices as per the topology. In this setup, ISIS routing protocol is configured on all the devices except the hosts, the host will be connected layer-2 dual-homed to the Leafs.

The “sh isis neighbor” and “show route isis” command can be used to verify that the adjacency is formed and the routes of all the Leafs and Spines are learnt via ISIS.


##Task 3: Configure and Verify ISIS Segment Routing as underlay:

Configure Segment Routing protocol under ISIS routing protocol which enables MPLS on all the non-passive ISIS interfaces. A prefix SID is associated with an IP prefix and is manually configured from the segment routing global block (SRGB) range of labels. It is configured under the loopback interface with the loopback address of the node as the prefix. The prefix SID is globally unique within the segment routing domain.

The Prefix-SID can be an absolute value or and indexed value. In this guide, we are configuring Prefix-SID as absolute value. ISIS Segment Routing is configured in the Fabric between Leafs and Spines. 

Following is a sample config to enable Segment Routing in the network. Similar config with prefix-SID that is unique for each device in the network, should be configured on other devices (as per the above diagram) to enable ISIS Segment Routing. In this config prefix-SID is enabled on the “loopback 0” interface of the devices.


    Spine-1:

    router isis 1
    address-family ipv4 unicast
      segment-routing mpls
     !
     interface Loopback0
      passive
      address-family ipv4 unicast
       prefix-sid absolute 16006
    !

    Spine-2:

    router isis 1
    address-family ipv4 unicast
      segment-routing mpls
     !
     interface Loopback0
      passive
      address-family ipv4 unicast
       prefix-sid absolute 16007
    !

Configure on all the Leafs “cef adjacency route override rib" cli command, to prefer CEF adjacency prefix created from the ARP entry over the RIB prefix entry.


	RP/0/RP0/CPU0:Leaf-1(config)#cef adjacency route override rib


Verify that all devices that have ISIS Segment Routing configured have advertised their prefix-SIDs. Also verify the prefix-SIDs are learnt and programmed in the forwarding plane on each device. 
This output is collected from Spines; we can see that the prefix-SID labels (identified by “Pfx”) of all the Leafs and other routers are learnt and programmed in the forwarding plane along with their outgoing interfaces.


    Spine-1:

    RP/0/RP0/CPU0:Spine-1#sh mpls forwarding 
    Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
    Label  Label       or ID              Interface                    Switched    
    ------ ----------- ------------------ ------------ --------------- ------------
    16001  Pop         SR Pfx (idx 1)     BE16         192.1.6.2       0           
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


    Leaf-1:

    RP/0/RP0/CPU0:Leaf-1#sh mpls forwarding 
    Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
    Label  Label       or ID              Interface                    Switched    
    ------ ----------- ------------------ ------------ --------------- ------------
    16002  16002       SR Pfx (idx 2)     BE16         192.1.6.1       0           
           16002       SR Pfx (idx 2)     BE17         192.1.7.1       0           
    16005  16005       SR Pfx (idx 5)     BE16         192.1.6.1       0         
           16005       SR Pfx (idx 5)     BE17         192.1.7.1       0         
    16006  Pop         SR Pfx (idx 6)     BE16         192.1.6.1       0          
    16007  Pop         SR Pfx (idx 7)     BE17         192.1.7.1       0           
    64000  Pop         SR Adj (idx 1)     BE16         192.1.6.1       0           
    64001  Pop         SR Adj (idx 3)     BE16         192.1.6.1       0           
    64002  Pop         SR Adj (idx 1)     BE17         192.1.7.1       0           
    64003  Pop         SR Adj (idx 3)     BE17         192.1.7.1       0


After configuring ISIS segment routing, verify that the underlay is capable of forwarding traffic using labels assigned by segment routing. 

Below output shows traceroute from Leaf-1 to Leaf-5 using the loopback address. Trace from Leaf-1 reaches Leaf-5 via Spines using label forwarding where Spine is the PHP for Leaf-5. 


    **Trace from Leaf-1 to Leaf-5:**

    RP/0/RP0/CPU0:Leaf-1#traceroute 5.5.5.5

    Type escape sequence to abort.
    Tracing the route to 5.5.5.5

     1  192.1.6.1 [MPLS: Label 16005 Exp 0] 2 msec  2 msec  1 msec 
     2  192.5.6.2 3 msec  *  4 msec 
    RP/0/RP0/CPU0:Leaf-1#



##Task 4: Configure the BGP-EVPN Control-Plane in the Fabric

MP-BGP with its various address families is used to transport specific reachability information in the network. BGP’s L2VPN-EVPN address family is capable of transporting tenant-aware/VRF-aware IP (Layer-3) and MAC (Layer-2) reachability information in MP-BGP. BGP EVPN provides the learnt information to all the devices within the network through a common control plane. BGP EVPN next-hops are going to be reachable via segment routing paths.
In this configuration guide to configure EVPN in the Fabric, we will configure iBGP EVPN, however eBGP EVPN can also be configured and is support on NCS 5500 routers. Spines are configured as the BGP EVPN Route Reflectors. Leaf-1, Leaf-2 and Leaf-5 will all be Route Reflector clients.


Configure Spines as RR for BGP EVPN address family.

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


Use the following configuration and apply it to configure the Leaf-1 Leaf-2 and Leaf-5 to form the BGP EVPN adjacency between Leafs and Route Reflectors. 


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


Use “sh bgp l2vpn evpn summary” cli command to verify the evpn neighborship between Route Reflectors and Leafs. Below output from the Spines show that the BGP EVPN neighborship is formed between the Leafs and the Route Reflectors and the control-plane is up. 



    Spine-1:

    RP/0/RP0/CPU0:Spine-1#sh bgp l2vpn evpn summary 
    BGP router identifier 6.6.6.6, local AS number 65001 

    Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
    Speaker               1          1          1          1           1           0

    Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
    1.1.1.1           0 65001       8       8        1    0    0 00:06:02          0
    2.2.2.2           0 65001       7       7        1    0    0 00:04:53          0
    5.5.5.5           0 65001       7       7        1    0    0 00:04:14          0


    Spine-2: 

    RP/0/RP0/CPU0:Spine-2#sh bgp l2vpn evpn summary 
    BGP router identifier 7.7.7.7, local AS number 65001

    Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
    Speaker               1          1          1          1           1           0

    Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
    1.1.1.1           0 65001       9      10        1    0    0 00:06:50          0
    2.2.2.2           0 65001       8       8        1    0    0 00:05:43          0
    5.5.5.5           0 65001       7       7        1    0    0 00:05:03          0


In this post we covered the configuration and verification of BGP-EVPN control-plane and ISIS-SR based forwarding plane. In the next post we will leverage the EVPN control-plane and ISIS-SR to provision EVPN based Layer-2 VPN stretch from Leaf-1/Leaf-2 to Leaf-5.