---
published: true
date: '2018-09-16 15:29 -0700'
title: BGP-EVPN Configuration on NCS 5500 part-3
author: Ahmad Bilal Siddiqui
position: top
excerpt: >-
  In this post we will cover the provisioning of BGP-EVPN based Layer-2 VPN
  service
tags:
  - iosxr
  - ncs 5500
  - evpn
  - NCS5500
---
{% include toc %}

# Topic: Configure BGP-EVPN based Layer-2 VPN service

In the [last post](https://xrdocs.io/cloud-scale-networking/tutorials/bgp-evpn-configuration-ncs-5500-part-2/), we configured the BGP-EVPN based Multi-homing of host/CE using EVPN Ethernet Segment. In this post, we will provision BGP-EVPN based Layer-2 VPN service between the Leafs. The EVPN Layer-2 service will enable forwarding between host-1, host-2 and host-5 which are part of the same subnet.

## Reference Topology:
![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/evpn-config/Host-connectivity.png?raw=true)

In this setup, Host-1 and Host-5 belong to the same subnet. Host-1 is dual-homed to Leaf-1 and Leaf-2 while Host-5 is single homed to the Leaf-5. Packets sourced from Host-1 for destination Host-5 will arrive to Leaf-1 or Leaf-2 based on the LAG’s hash calculation. On Leaf the lookup will be performed for destination Host-5 MAC address. Host-5’s MAC address will be learnt on Leaf-1 and Leaf-2 via EVPN control-plane. After the lookup, the traffic will be forwarded to the Host-5 MAC address using EVPN service label and transport label to reach to Leaf-5. 

![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/evpn-config/EVPN-based-L2-VPN-service.png?raw=true)

### Task 1: Configure Host-1 and Host-5 IP address 

Host-1 and Host-5 will be part of the same subnet to communicate over layer-2 stretch. Host-1 is connected dual-homed to uplink Leafs via LACP link aggregation and Host-5 is connected single-homed to Leaf-5. Configure IP address on Host-1’s and Host-5 as follows.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Host-1

    interface Bundle-Ether1
     description "Bundle to Leaf-1/2"
     ipv4 address 10.0.0.10 255.255.255.0
     !

    Host-5

    interface TenGigE0/0/2/0
     description "Link to Leaf-5"
     ipv4 address 10.0.0.50 255.255.255.0
     !
</code>
</pre>
</div>


### Task 2: Configure Layer-2 interfaces and Bridge Domain on Leafs

Configure layer-2 interfaces with dot1q encapsulation for VLAN 10 on Leaf-1 and Leaf-2. Use the following configuration for both Leaf-1, Leaf-2 and Leaf-5.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1 and Leaf-2

    interface Bundle-Ether 1.10 l2transport
     encapsulation dot1q 10
     rewrite ingress tag pop 1 symmetric
    !

    Leaf-5

    interface TenGigE0/0/0/47.10 l2transport
     encapsulation dot1q 10
     rewrite ingress tag pop 1 symmetric
    !
</code>
</pre>
</div>

Configure Bridge domain for the VLAN and add the VLAN tagged interfaces to the bridge-domain. Configure the following on Leaf-1, Leaf-2 and Leaf-5.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1 and Leaf-2

    l2vpn
     bridge group bg-1
      bridge-domain bd-10
       interface Bundle-Ether 1.10
       !
    !


    Leaf-5

    l2vpn
     bridge group bg-1
      bridge-domain bd-10
       interface TenGigE0/0/0/47.10
       !
    ! 
</code>
</pre>
</div>

Verify that the bridge-domain and the related attachment circuits are up. Following output shows that the bridge-domain bd-10’s state is ‘up’, its attachment circuit is ‘up’.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1

    RP/0/RP0/CPU0:Leaf-1#show l2vpn bridge-domain bd-name bd-10
    Legend: pp = Partially Programmed.
    Bridge group: bg-1, bridge-domain: bd-10, id: 0, <mark>state: up,</mark> ShgId: 0, MSTi: 0
      Aging: 300 s, MAC limit: 64000, Action: none, Notification: syslog
      Filter MAC addresses: 0
      <mark>ACs: 1 (1 up),</mark> VFIs: 0, PWs: 0 (0 up), PBBs: 0 (0 up), VNIs: 0 (0 up)
      List of ACs:
        BE1.10, state: up, Static MAC addresses: 0
      List of Access PWs:
      List of VFIs:
      List of Access VFIs:


    Leaf-5

    RP/0/RP0/CPU0:Leaf-5#show l2vpn bridge-domain bd-name bd-10
    Legend: pp = Partially Programmed.
    Bridge group: bg-1, bridge-domain: bd-10, id: 0, <mark>state: up,</mark> ShgId: 0, MSTi: 0
      Aging: 300 s, MAC limit: 64000, Action: none, Notification: syslog
      Filter MAC addresses: 0
      <mark>ACs: 1 (1 up),</mark> VFIs: 0, PWs: 0 (0 up), PBBs: 0 (0 up), VNIs: 0 (0 up)
      List of ACs:
        Te0/0/0/47.10, state: up, Static MAC addresses: 0
      List of Access PWs:
      List of VFIs:
      List of Access VFIs:
    RP/0/RP0/CPU0:Leaf-5#
</code>
</pre>
</div>

So far, we have configured local bridging on the Leafs and connected them to the hosts for vlan 10 tagged data. We verified that the local bridging and attachment circuits are ‘up’. In order for Host-1 to communicate to Host-5 via layer-2, we need to configure layer-2 stretch/service between the Leafs to which Hosts are connected. 

The layer-2 service/stretch across the Leafs is offered by configuring EVPN EVI (EVPN Instance). EVI allows the layer-2 to be stretched via MP-BGP EVPN control-plane across multiple participating Leafs/PEs. An EVI is configured on a per layer-2 bridge basis across Leafs/PEs. Each EVI has a unique route distinguisher and one or more route targets.

For Layer-2 VPN use case, we are stretching the layer-2 between Leaf-1, Leaf-2 and Leaf-5. Therefore, we will provision Layer-2 VPN service by configure EVI on all three leafs.


### Task 3: Configure EVPN EVI on Leaf-1, Leaf-2 for VLAN 10

First we will configure the EVI on Leaf-1 and Leaf-2, then we will verify that the Ethernet Segment for vlan 10 tagged data is up. 

Configure EVI in EVPN config on Leaf-1 and Leaf-2. Also assign the route-target values for the EVI related network to get advertised and received via BGP EVPN control-plane. Advertise-mac keyword is used to advertise the MAC addresses in EVI to other Leafs part of EVI via BGP EVPN. 
 
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1 and Leaf-2

    evpn
     <mark>evi 10</mark>
      bgp
       route-target import 1001:11
       route-target export 1001:11
      !
      <mark>advertise-mac</mark>
      !
     source interface loopback 0
     !
</code>
</pre>
</div>

Associate the EVI to bridge-domain for VLAN 10, this is where the attachment-circuit/host is connected to.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    l2vpn
     bridge group bg-1
      bridge-domain bd-10
       <mark>evi 10</mark>
       !
      !
</code>
</pre>
</div>


As we have now configured layer-2 service with EVI for Bridge-domain 10, lets verify the Ethernet Segment status to see that the multi-homing is operational for Bridge-domain 10 forwarding. 

Observe in the below output that for Ethernet-segment bundle interface ‘BE1’, there are two next-hops. The next-hops represent each Leaf-1 and Leaf-2 forming Leaf pair for Ethernet Segment. Also in below output we can see that Ethernet-segment state is ‘Up’ and all-active multi-homing is operational. We have one forwarder which is VLAN 10 and Leaf-1 is the elected designated forwarded (DF) for it. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1 

    RP/0/RP0/CPU0:Leaf-1#show evpn ethernet-segment detail 
 
    Ethernet Segment Id      Interface                          Nexthops            
    ------------------------ ---------------------------------- --------------------
    0011.1111.1111.1111.1111 BE1                                <mark>1.1.1.1</mark>
                                                                <mark>2.2.2.2</mark>
      ES to BGP Gates   : Ready
      ES to L2FIB Gates : Ready
      Main port         :
         Interface name : Bundle-Ether1
         Interface MAC  : 00bc.601c.d0da
         IfHandle       : 0x08000044
         <mark>State          : Up</mark>
         Redundancy     : Not Defined
      ESI type          : 0
         Value          : 11.1111.1111.1111.1111
      ES Import RT      : 1111.1111.1111 (Local)
      Source MAC        : 0000.0000.0000 (N/A)
      Topology          :
         <mark>Operational    : MH, All-active</mark>
         Configured     : All-active (AApF) (default)
      Service Carving   : Auto-selection
      Peering Details   : 1.1.1.1[MOD:P:00] 2.2.2.2[MOD:P:00]
      Service Carving Results:
         <mark>Forwarders     : 1</mark>
         Permanent      : 0
         Elected        : 1
         Not Elected    : 0
      MAC Flushing mode : STP-TCN
      Peering timer     : 3 sec [not running]
      Recovery timer    : 30 sec [not running]
      Carving timer     : 0 sec [not running]
      Local SHG label   : 64007
      Remote SHG labels : 1
                  64008 : nexthop 2.2.2.2

    RP/0/RP0/CPU0:Leaf-1#
</code>
</pre>
</div>


With the following CLI command we can verify that the MAC address of Host-1 is being learnt on Leaf-1 and Leaf-2. MAC address of Host-5 will be learnt on Leaf-1 and Leaf-2 after we configure EVI on Leaf-5 for VLAN 10 layer-2 stretch. 
<div class="highlighter-rouge">
<pre class="highlight">
<code>
	Leaf-1 

    RP/0/RP0/CPU0:Leaf-1#show l2route evpn mac all 
    Sat Sep  1 22:45:53.336 UTC
    Topo ID  Mac Address    Producer    Next Hop(s)                             
    -------- -------------- ----------- ----------------------------------------
    0        <mark>6c9c.ed6d.1d8b</mark> LOCAL       Bundle-Ether1.10
    RP/0/RP0/CPU0:Leaf-1#


    Leaf-2

    RP/0/RP0/CPU0:Leaf-2#show l2route evpn mac all 
    Sat Sep  1 22:49:43.498 UTC
    Topo ID  Mac Address    Producer    Next Hop(s)                             
    -------- -------------- ----------- ----------------------------------------
    0        <mark>6c9c.ed6d.1d8b</mark> L2VPN       Bundle-Ether1.10                        
    RP/0/RP0/CPU0:Leaf-2#
</code>
</pre>
</div>

### Task 4: Configure EVPN EVI on Leaf-5 for VLAN 10 
<div class="highlighter-rouge">
<pre class="highlight">
<code>
    On Leaf-5

    evpn
     <mark>evi 10</mark>
      bgp
       route-target import 1001:11
       route-target export 1001:11
      !
      <mark>advertise-mac</mark>
      !
     source interface loopback 0
     !
</code>
</pre>
</div>


Associate the EVI to bridge-domain for VLAN 10, this is where the attachment-circuit/host is connected to.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    l2vpn
     bridge group bg-1
      bridge-domain bd-10
       <mark>evi 10</mark>
       !
      !
</code>
</pre>
</div>

### Task 5: Verify EVPN EVI and Layer-2 Stretch between the Leaf-1, Leaf-2 and Leaf-5

We have configured the Layer-2 stretch between Leaf-1, Leaf-2 and Leaf-5 using EVPN EVI. In the next steps lets verify the layer-2 connectivity is up and we can reach from one host to another via layer-2. “show evpn evi detail” cli command shows the configured EVI and its associated bridge-domain. It also shows the route-target import and export values as shown in the below output.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    RP/0/RP0/CPU0:Leaf-1#show evpn evi detail
    Sat Sep  1 23:13:01.611 UTC

    VPN-ID     Encap  Bridge Domain                Type               
    ---------- ------ ---------------------------- -------------------
    10         MPLS   bd-10                        EVPN               
       Stitching: Regular
       Unicast Label  : 64004
       Multicast Label: 64120
       Flow Label: N
       Control-Word: Enabled
       Forward-class: 0
       Advertise MACs: Yes
       Advertise BVI MACs: No
       Aliasing: Enabled
       UUF: Enabled
       Re-origination: Enabled
       Multicast source connected: No

       Statistics:
         Packets            Sent                 Received
           Total          : 0                    0                   
           Unicast        : 0                    0                   
           BUM            : 0                    0                   
         Bytes              Sent                 Received
           Total          : 0                    0                   
           Unicast        : 0                    0                   
           BUM            : 0                    0                   
       RD Config: none
       RD Auto  : (auto) 1.1.1.1:10
       RT Auto  : 65001:10
       Route Targets in Use           Type                 
       ------------------------------ ---------------------
       1001:11                        Import               
       1001:11                        Export               

    RP/0/RP0/CPU0:Leaf-1#
</code>
</pre>
</div>

Ping from Host-1 to Host-5 and verify that the Hosts are reachable. We can see in the below output that that Host-1 can ping Host-5. Also, below output shows that the MAC address for Host-5 is learnt on Leaf-1 and Leaf-2. Similarly, we are learning the MAC address of Host-1 on Leaf-5. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-1
    
    RP/0/RP0/CPU0:Leaf-1#show l2route evpn mac all 
    Sat Sep  1 22:53:57.880 UTC
    Topo ID  Mac Address    Producer    Next Hop(s)                             
    -------- -------------- ----------- ----------------------------------------
    0        <mark>6c9c.ed6d.1d8b</mark> LOCAL       Bundle-Ether1.10                        
    0        a03d.6f3d.5443 L2VPN       5.5.5.5/64004/ME                        
    RP/0/RP0/CPU0:Leaf-1#

    Leaf-2

    RP/0/RP0/CPU0:Leaf-2#show l2route evpn mac all 
    Sat Sep  1 23:00:03.487 UTC
    Topo ID  Mac Address    Producer    Next Hop(s)                             
    -------- -------------- ----------- ----------------------------------------
    0        <mark>6c9c.ed6d.1d8b</mark> L2VPN       Bundle-Ether1.10                        
    0        a03d.6f3d.5443 L2VPN       5.5.5.5/64004/ME                        
    RP/0/RP0/CPU0:Leaf-2#

    Leaf-5

    RP/0/RP0/CPU0:Leaf-5#show l2route evpn mac all 
    Sat Sep  1 23:00:03.785 UTC
    Topo ID  Mac Address    Producer    Next Hop(s)                             
    -------- -------------- ----------- ----------------------------------------
    0        <mark>6c9c.ed6d.1d8b</mark> L2VPN       64005/I/ME                              
    0        a03d.6f3d.5443 LOCAL       TenGigE0/0/0/47.10                      
    RP/0/RP0/CPU0:Leaf-5#
</code>
</pre>
</div>


We can verify the BGP EVPN control-plane to verify the various routes and mac addresses are advertised and learnt. 

In the below output from Leaf-1 we can see the MAC address of Host-1 and Host-5 are being learnt under their respective route distinguishers. MAC addresses are advertised using EVPN Route-Type-2.
 
Example of Host-1 MAC learnt **([2][0][48][6c9c.ed6d.1d8b][0]/104)** 

The route distinguisher value is comprised of **router-id:EVI eg. 1.1.1.1:10, 2.2.2.2:10** which are highlighted below.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-5

    RP/0/RP0/CPU0:Leaf-5#show bgp l2vpn evpn rd 1.1.1.1:10

    Status codes: s suppressed, d damped, h history, * valid, > best
                  i - internal, r RIB-failure, S stale, N Nexthop-discard
    Origin codes: i - IGP, e - EGP, ? - incomplete
       Network            Next Hop            Metric LocPrf Weight Path
    <mark>Route Distinguisher: 1.1.1.1:10</mark>
    *>i[1][0011.1111.1111.1111.1111][0]/120
                          1.1.1.1                       100      0 i
    * i                   1.1.1.1                       100      0 i
    *>i[2][0][48][6c9c.ed6d.1d8b][0]/104
                          1.1.1.1                       100      0 i
    * i                   1.1.1.1                       100      0 i
    *>i[3][0][32][1.1.1.1]/80
                          1.1.1.1                       100      0 i
    * i                   1.1.1.1                       100      0 i

    Processed 3 prefixes, 6 paths
    RP/0/RP0/CPU0:Leaf-5#



    RP/0/RP0/CPU0:Leaf-5#show bgp l2vpn evpn rd 2.2.2.2:10

    Status codes: s suppressed, d damped, h history, * valid, > best
                  i - internal, r RIB-failure, S stale, N Nexthop-discard
    Origin codes: i - IGP, e - EGP, ? - incomplete
       Network            Next Hop            Metric LocPrf Weight Path
    <mark>Route Distinguisher: 2.2.2.2:10</mark>
    *>i[1][0011.1111.1111.1111.1111][0]/120
                          2.2.2.2                       100      0 i
    * i                   2.2.2.2                       100      0 i
    *>i[2][0][48][6c9c.ed6d.1d8b][0]/104
                          2.2.2.2                       100      0 i
    * i                   2.2.2.2                       100      0 i
    *>i[3][0][32][2.2.2.2]/80
                          2.2.2.2                       100      0 i
    * i                   2.2.2.2                       100      0 i

    Processed 3 prefixes, 6 paths
    RP/0/RP0/CPU0:Leaf-5#
</code>
</pre>
</div>

CLI command “show evpn evi vpn-id 10 mac” can be used to verify the MAC address and Host IP addresses being learnt related to the EVI. In the following output of EVI table from Leaf-5, we can see that we are learning MAC address of Host-1 via EVI 10 on Leaf-5. We can reach to Host-1 MAC address either via next-hop 1.1.1.1 of Leaf-1 or 2.2.2.2 which is Leaf-2. We can run the same command on Leaf-1 and Leaf-2 for verification. 

<div class="highlighter-rouge">
<pre class="highlight">
<code>
    Leaf-5

    RP/0/RP0/CPU0:Leaf-5#show evpn evi vpn-id 10 mac
    Sat Sep  1 23:24:00.808 UTC

    VPN-ID     Encap  MAC address    IP address       Nexthop                                 Label   
    ---------- ------ -------------- ---------------------------------------- -----------------------------
    10         MPLS   <mark>6c9c.ed6d.1d8b ::               1.1.1.1</mark>                                64004   
    10         MPLS   <mark>6c9c.ed6d.1d8b ::               2.2.2.2</mark>                                64004   
    10         MPLS   a03d.6f3d.5443 ::               TenGigE0/0/0/47.10                      64004   
    RP/0/RP0/CPU0:Leaf-5#
</code>
</pre>
</div>


We are only seeing MAC address and not IP address of the Host in the below out. This is because we configured only Layer-2 service between the Leafs. Once we configure IRB with Host-routing, we will start advertising MAC + IP of the host via EVPN Route-Type-2 and will be able to see IP address in the below table as well as in Leaf’s routing table.  

In the next post, we will cover EVPN IRB and Host-Routing configuration in detail.
