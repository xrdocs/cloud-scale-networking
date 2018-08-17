---
published: false
date: '2018-08-17 00:28 +0200'
title: NCS5500 FIB Programming Speed
author: Nicolas Fevrier
excerpt: >-
  In this post, we focus on the speed the system can program routes in RP and in
  hardware
---
{% include toc icon="table" title="NCS5500 FIB Programming Speed" %} 

You can find more content related to NCS5500 including routing memory management, VRF, URPF, ACLs, Netflow following this [link](https://xrdocs.io/cloud-scale-networking/tutorials/).

## Programming Speed

In this post, we will measure the time it takes to learn routes in the RIB and in the FIB.

The first one exists in the Route Processor and will be fed by a BGP process.  
The second one exists in multiple places, but to simplify the discussion, we will measure what is actually programmed in the NPU database.  

We often hear that "Merchant Silicon systems program prefixes slowers than other products" but clearly this assertion is not based on facts and we will debunk it with this post and video.

### Video demo

Let's get started with a video we recorded and published on youtube.  

[![NCS5500 Programming Speed](https://img.youtube.com/vi/o4pUTniOuRY/0.jpg)](https://www.youtube.com/watch?v=o4pUTniOuRY){: .align-center}

In this demo, we advertised 1,2000,000 IPv4 routes to our system under test:  
300K IPv4/22, 300K IPv4/23 and 600K IPv4/25

<div class="highlighter-rouge">
<pre class="highlight">
<code> 
router bgp 1000
bgp_id 192.168.100.151
neighbor 192.168.100.200 remote-as 100
neighbor 192.168.100.200 update-source 192.168.100.151
capability ipv4 unicast
network 1 11.0.0.0/23 300000
nexthop 1 192.168.22.1
network 2 101.0.0.0/25 600000
nexthop 2 192.168.22.1
network 3 171.0.0.0/22 300000
nexthop 3 192.168.22.1
capability refresh
</code>
</pre>
</div>


### Test methodology

The system we will use for this demo is a chassis with a 36x 100G ports "Scale". That means, it's based on Jericho+ chipset with a new generation external TCAM. Since we are using IOS XR 6.3.2 or 6.5.1, all routes (IPv4 and IPv6) are stored on the eTCAM, regarless their prefix length.

The speed a router learns BGP routes is directly dependant on the speed the neighbor is able to advertise these prefixes. Since BGP is based on TCP, all messages are ack'd and the local process can request to slow down for any reason. That's why we thought it woud not be relevant to use a route generator for this test. Or at least, we didn't want the router under test to be directly peered to the route generator.

We decided to use an intermediate system of the same kind, for instance an NCS55A1-24H. This system will receive the BGP table from our route generator. When all the routes will be received in this intermediate system, we will enable the BGP session to the system under test.

That way, the routes are advertised from a real router BGP stack and the results are representing what you could expect in your production environment.

**Step 0**: "Before the test"





### Test with real internet table

RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#sh bgp sum
Thu Aug 16 15:44:18.461 PDT
BGP router identifier 1.1.1.22, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 23817074
BGP main routing table version 23817074
BGP NSR Initial initsync version 1200006 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker        23817074   23817074   23817074   23817074    23817074           0

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.22.1      0   100  164033   15246        0    0    0 00:26:05 Idle (Admin)
192.168.100.151   0  1000 1241354   49602 23817074    0    0 00:05:14     751657

RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#sh dpa resources iproute loc 0/0/CPU0
Thu Aug 16 15:44:25.163 PDT

"iproute" DPA Table (Id: 24, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution
Prefix   Actual       Prefix   Actual
 /0       1            /1       0
 /2       0            /3       0
 /4       1            /5       0
 /6       0            /7       0
 /8       15           /9       13
 /10      35           /11      106
 /12      285          /13      550
 /14      1066         /15      1880
 /16      13419        /17      7773
 /18      13636        /19      25026
 /20      38261        /21      43073
 /22      80751        /23      67073
 /24      376982       /25      567
 /26      2032         /27      4863
 /28      15599        /29      16868
 /30      41735        /31      52
 /32      15
                          NPU ID: NPU-0           NPU-1
                          In Use: 751677          751677
                 Create Requests
                           Total: 12246542        12246542
                         Success: 12246542        12246542
                 Delete Requests
                           Total: 11494865        11494865
                         Success: 11494865        11494865
                 Update Requests
                           Total: 75808           75808
                         Success: 75804           75804
                    EOD Requests
                           Total: 0               0
                         Success: 0               0
                          Errors
                     HW Failures: 301856          301856
                Resolve Failures: 0               0
                 No memory in DB: 0               0
                 Not found in DB: 0               0
                    Exists in DB: 0               0

RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#conf
Thu Aug 16 15:45:13.118 PDT
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config)#
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config)#router bgp 100
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config-bgp)# neighbor 192.168.22.1
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config-bgp-nbr)#  remote-as 100
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config-bgp-nbr)#  shutdown
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config-bgp-nbr)#no shut
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config-bgp-nbr)#commit
Thu Aug 16 15:45:22.121 PDT
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2(config-bgp-nbr)#end
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#rollback configuration last 1
Thu Aug 16 15:50:30.938 PDT

Loading Rollback Changes.
Loaded Rollback Changes in 1 sec
Committing.
5 items committed in 1 sec
Updating.
Updated Commit database in 1 sec
Configuration successfully rolled back 1 commits.
RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#sh bgp sum
Thu Aug 16 15:51:30.181 PDT
BGP router identifier 1.1.1.22, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 23817074
BGP main routing table version 23817074
BGP NSR Initial initsync version 1200006 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.


Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker        23817074   23817074   23817074   23817074    23817074           0

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.22.1      0   100  164041  179174        0    0    0 00:00:56 Idle (Admin)
192.168.100.151   0  1000 1241361   49609 23817074    0    0 00:12:26     751657

RP/0/RP0/CPU0:NCS55A1-24H-6.3.2#

