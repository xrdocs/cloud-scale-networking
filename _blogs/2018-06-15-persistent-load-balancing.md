---
published: true
date: '2018-06-15 14:41 -0400'
title: Persistent Load Balancing or "Sticky ECMP"
author: Xander Thuijs
excerpt: >-
  Traditional ECMP or equal cost multipath loadbalances traffic over a number of
  available paths towards a destination. When one path fails, the traffic gets
  re-shuffled over the available number of paths.
position: top
---


{% include toc icon="table" title='Persistent Load Balancing or "Sticky ECMP"' %}  
{% include base_path %}

## Introduction

This document applies to NCS5500 and ASR9000 routers and has been verified as such.
{: .notice--info}

Traditional ECMP or equal cost multipath loadbalances traffic over a number of available paths towards a destination. When one path fails, the traffic gets re-shuffled over the available number of paths.

This means that a flow that was before taking path "1", could now be taking path "3" although only path "2" failed.

This reshifting occurs because the hash of althogh the flow remains the same resulting in the same bucket, but the bucket may get reassigned to a new path.


To understand flows, buckets and traditional ECMP a bit better, you could reference the Loadbalancing [Architecture document](https://supportforums.cisco.com/t5/service-providers-documents/asr9000-xr-load-balancing-architecture-and-characteristics/ta-p/3124809) and consult the [Cisco Live ID 2904](https://www.ciscolive.com/global/on-demand-library/?search=2904&search.event=ciscoliveus2017&search.event=ciscoliveus2015&search.event=ciscoliveus2014#/) from Las Vegas 2017.

 
 
While this flow redistribution is not a problem in traditional core networks, because the end to end connectivity is preserved and the user would not experience any situation from it, in data center loadbalancing this can be a problem.


![](https://kxiwq67737.i.lithium.com/t5/image/serverpage/image-id/9922i57D745C0A47E7B4F/image-size/large?v=1.0&px=999)


### Datacenter loadbalancing
 
 
This rehashing as mentioned can be troublesome in data center environments where many servers advertise a "service prefix" to a loadbalancer/gateway in a sort of "anycast" way.

This results in a user connecting with its l3/l4 tupple would be delegated to one particular server for a session.

If for whatever reason a server fails, we don't want the established session to a server to be rehashed to a new server as that will reset the tCP connection sicne that new server may have no clue (~socket :) about this session it just got a packet for.

 

To visualize:

![](https://kxiwq67737.i.lithium.com/t5/image/serverpage/image-id/9924i411B1CA9B4A9B556/image-size/large?v=1.0&px=999)


Persistent Loadbalancing or Sticky ECMP defines a prefix in such a way that we dont rehash flows on existing paths and only replace those bucket assignments of the failed server.

 
The good thing is that established sessions to servers wont get rehashed.

The downside of this is that you could see more load on one server then another now. (Traditional ECMP would try to achieve equal spread, at the cost of that rehashing).

## Implementation details

- How to map prefixes for sticky ECMP ?

Use an RPL to define prefixes that require persistent load balancing. User would match some BGP community to set sticky ecmp flag

- What happens when a path in an ECMP goes down ?

In FIB each prefix has a path list, say for example a prefix ‘X’ has a path list (p1, p2, p3) and when a path say ‘p2’ fails with sticky ECMP enabled new path list become (p1, p1, p3), instead of the default rehash logic, which results (p1, p3, p1)

- What happens when a link comes back ?

There are 2 modes of operation:

**DEFAULT**: No rehashing is done and the link will not be utilized until one of the

following happens, which results a complete recalculation of paths.

 - New path addition to ECMP.
 - User driven clear operation using “clear route” command.
 
**CONFIGURABLE**: Auto recovery. If the server comes back or the path gets reenabled, we automatically reshuffle the sessions, this will result in sessions that were moved from the failed path to a new server will now be rehashed BACK to the original server that got back online, this will result in session disruption ONLY for those sessions.

There is no one size fits all answer here hence we provide the 2 options:

manual recovery or automatic recovery, with both pros and cons.

## Configuration


 Now that you're all excited about this new functionality, you want to try it out right? here is the configuration sequence on how to establish it:

 

First define the route policy that will direct which prefixes are to be marked as sticky.


```
route-policy sticky-ecmp
  if destination in (192.168.3.0/24) then
    set load-balance ecmp-consistent
  else
    pass
  endif
end-policy
```

Apply that route policy to BGP through the table-policy directive:


<div class="highlighter-rouge">
<pre class="highlight">
<code>
router bgp 7500
 address-family ipv4 unicast
  <b>table-policy sticky-ecmp</b>
  maximum-paths ebgp 64
  maximum-paths ibgp 32
  ! need to have multipath enabled obviously
</code>
</pre>
</div>

That's it! 
{: .notice--success}

### Verification of operation
 
Let's verify the CEF display before a failure occurred:


Show cef <prefix> detail


<div class="highlighter-rouge">
<pre class="highlight">
<code>
 LDI Update time Sep  5 11:22:38.201
   via 10.1.0.1/32, 3 dependencies, recursive, bgp-multipath [flags 0x6080]
    <b>path-idx 0</b> NHID 0x0 [0x57ac4e74 0x0]
    next hop 10.1.0.1/32 via 10.1.0.1/32
   via 10.2.0.1/32, 3 dependencies, recursive, bgp-multipath [flags 0x6080]
    <b>path-idx 1</b> NHID 0x0 [0x57ac4a74 0x0]
    next hop 10.2.0.1/32 via 10.2.0.1/32
   via 10.3.0.1/32, 3 dependencies, recursive, bgp-multipath [flags 0x6080]
    <b>path-idx 2</b> NHID 0x0 [0x57ac4f74 0x0]
    next hop 10.3.0.1/32 via 10.3.0.1/32

    Load distribution <span style="background-color: #FDD7E4">(persistent)</span>: 0 1 2 (refcount 1)
    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/0    10.1.0.1      
    1     Y   GigabitEthernet0/0/0/1    10.2.0.1      
    2     Y   GigabitEthernet0/0/0/2    10.3.0.1  
</code>
</pre>
</div>

We see 3 paths identified with 3 next hops (10.1/2/3.0.1) via 3 different gig interfaces. We can also see here that the stickiness is enabled through the "persistent" keyword.

After a path failure, in this example we brought gig 0/0/0/1 down:

Show cef <prefix> detail


<div class="highlighter-rouge">
<pre class="highlight">
<code>

 LDI Update time Sep  5 11:23:13.434
   via 10.1.0.1/32, 3 dependencies, recursive, bgp-multipath [flags 0x6080]
    path-idx 0 NHID 0x0 [0x57ac4e74 0x0]
    next hop 10.1.0.1/32 via 10.1.0.1/32
   via 10.3.0.1/32, 3 dependencies, recursive, bgp-multipath [flags 0x6080]
    path-idx 1 NHID 0x0 [0x57ac4f74 0x0]
    next hop 10.3.0.1/32 via 10.3.0.1/32

    Load distribution <b>(persistent)</b> : 0 1 2 (refcount 1)
    Hash  OK  Interface                 Address
    0     Y   GigabitEthernet0/0/0/0    10.1.0.1      
    <b>1*    Y   GigabitEthernet0/0/0/0    10.1.0.1</b>       
    2     Y   GigabitEthernet0/0/0/2    10.3.0.1
    
    
</code>
</pre>
</div>

Notice the replacement of bucket 1 with gig 0/0/0/0 and the "*" denoting that this path is a replacement as it took a hit before.

We keep the bucket sequence in tact, we jsut replace it with an available path index.

Note that this will keep this way irrespective of gig0/0/0/1 coming back alive.

 

To recover the paths and put gig0/0/0/1 back in service on the hashing use:

```
clear route <prefix>
```

### Auto recovery
 

To enable the auto recovery, configure

```
cef consistent-hashing auto-recovery
```

A full trace sequence is given here with some show commands and verification:


<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RSP0/CPU0:PE1#sh run | i cef

Building configuration...

 bgp graceful-restart

<b>cef consistent-hashing auto-recovery</b>

 

RP/0/RSP0/CPU0:PE1#sho cef 192.168.3.0/24 detail 

192.168.3.0/24, version 674, internal 0x5000001 0x0 (ptr 0x722448fc) [1], 0x0 (0x0), 0x0 (0x0)

 Updated Nov  4 08:14:21.731

 Prefix Len 24, traffic index 0, precedence n/a, priority 4

 BGP Attribute: id: 0x6, Local id: 0x2, Origin AS: 0, Next Hop AS: 0

 ASPATH   :  

 Community: 

 

  gateway array (0x72ce5574) reference count 1, flags 0x2010, source rib (7), 0 backups

                [1 type 3 flags 0x48441 (0x72180850) ext 0x0 (0x0)]

  LW-LDI[type=0, refc=0, ptr=0x0, sh-ldi=0x0]

  gateway array update type-time 1 Nov  4 08:14:21.731

 LDI Update time Jan  1 21:23:30.335

 

  <b>Level 1 - Load distribution (consistent): 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15</b>

  [0] via 12.4.18.2/32, recursive

  [1] via 12.5.19.2/32, recursive

  [2] via 12.6.20.2/32, recursive

  [3] via 12.101.45.2/32, recursive

  [4] via 12.104.46.2/32, recursive

  [5] via 12.105.47.2/32, recursive

  [6] via 12.106.49.2/32, recursive

  [7] via 12.107.43.2/32, recursive

  [8] via 12.111.48.2/32, recursive

  [9] via 12.112.44.2/32, recursive

  [10] via 12.122.18.2/32, recursive

  [11] via 12.150.16.2/32, recursive

  [12] via 12.151.17.2/32, recursive

  [13] via 12.152.9.2/32, recursive

  [14] via 12.153.23.2/32, recursive

  [15] via 12.154.0.2/32, recursive
 

RP/0/RSP0/CPU0:PE1#cle counters a

Clear "show interface" counters on all interfaces [confirm]

RP/0/RSP0/CPU0:Jan  1 21:25:20.059 PDT: statsd_manager_g[1167]: %MGBL-IFSTATS-6-CLEAR_COUNTERS : Clear counters on all interfaces 

<b>RP/0/RSP0/CPU0:PE1#LC/0/1/CPU0:Jan  1 21:25:28.050 PDT: ifmgr[215]: %PKT_INFRA-LINK-3-UPDOWN : Interface TenGigE0/1/0/5/0, changed state to Down</b>

LC/0/1/CPU0:Jan  1 21:25:28.050 PDT: ifmgr[215]: %PKT_INFRA-LINEPROTO-5-UPDOWN : Line protocol on Interface TenGigE0/1/0/5/0, changed state to Down 
 

RP/0/RSP0/CPU0:PE1#show int tenGigE 0/1/0/5/0 ac

TenGigE0/1/0/5/0

  Protocol              Pkts In         Chars In     Pkts Out        Chars Out

  IPV4_UNICAST                1               59        98123         96355844

RP/0/RSP0/CPU0:PE1#cle counters 

Clear "show interface" counters on all interfaces [confirm]

RP/0/RSP0/CPU0:Jan  1 21:25:38.896 PDT: statsd_manager_g[1167]: %MGBL-IFSTATS-6-CLEAR_COUNTERS : Clear counters on all interfaces 

RP/0/RSP0/CPU0:PE1#

RP/0/RSP0/CPU0:PE1#LC/0/1/CPU0:Jan  1 21:25:43.353 PDT: pfm_node_lc[302]: %PLATFORM-CPAK-2-LANE_0_LOW_RX_POWER_ALARM : Set|envmon_lc[163927]|0x1005005|TenGigE0/1/0/5/0 

 

RP/0/RSP0/CPU0:PE1#LC/0/1/CPU0:Jan  1 21:25:50.110 PDT: ifmgr[215]: %PKT_INFRA-LINK-3-UPDOWN : Interface TenGigE0/1/0/5/0, changed state to Up 

<b>LC/0/1/CPU0:Jan  1 21:25:50.110 PDT: ifmgr[215]: %PKT_INFRA-LINEPROTO-5-UPDOWN : Line protocol on Interface TenGigE0/1/0/5/0, changed state to Up </b>

 

RP/0/RSP0/CPU0:PE1#show int tenGigE 0/1/0/5/0 ac

TenGigE0/1/0/5/0

  Protocol              Pkts In         Chars In     Pkts Out        Chars Out

  ARP                         1               60            1               42

 

 

RP/0/RSP0/CPU0:PE1#show int tenGigE 0/1/0/5/0 ac

TenGigE0/1/0/5/0

  Protocol              Pkts In         Chars In     Pkts Out        Chars Out
  <b>IPV4_UNICAST                0                0        24585         24142470</b>
  ARP                         1               60            1               42

 
RP/0/RSP0/CPU0:PE1#sho cef 192.168.3.0/24 detail 

192.168.3.0/24, version 674, internal 0x5000001 0x0 (ptr 0x722448fc) [1], 0x0 (0x0), 0x0 (0x0)

 Updated Nov  4 08:14:21.731

 Prefix Len 24, traffic index 0, precedence n/a, priority 4

 BGP Attribute: id: 0x6, Local id: 0x2, Origin AS: 0, Next Hop AS: 0

 ASPATH   :  

 Community: 

 

  gateway array (0x72ce5fc4) reference count 1, flags 0x2010, source rib (7), 0 backups

                [1 type 3 flags 0x48441 (0x721807d0) ext 0x0 (0x0)]

  LW-LDI[type=0, refc=0, ptr=0x0, sh-ldi=0x0]

  gateway array update type-time 1 Nov  4 08:14:21.731

 LDI Update time Jan  1 21:25:53.128

 

  Level 1 - Load distribution (consistent): 0 1 2 3 4 5 6 7 8 9 10 11 12 13 14 15

  [0] via 12.4.18.2/32, recursive

  [1] via 12.5.19.2/32, recursive

  [2] via 12.6.20.2/32, recursive

  [3] via 12.101.45.2/32, recursive

  [4] via 12.104.46.2/32, recursive

  [5] via 12.105.47.2/32, recursive

  [6] via 12.106.49.2/32, recursive

  [7] via 12.107.43.2/32, recursive

  [8] via 12.111.48.2/32, recursive

  [9] via 12.112.44.2/32, recursive

  [10] via 12.122.18.2/32, recursive

  [11] via 12.150.16.2/32, recursive

  [12] via 12.151.17.2/32, recursive

  [13] via 12.152.9.2/32, recursive

  [14] via 12.153.23.2/32, recursive

  [15] via 12.154.0.2/32, recursive
  
</code>
</pre>
</div>


## Restrictions and limitations

* Sticky load balancing is more resource intensive operation so it is not advised to enable it for all prefixes.
* Only supported for BGP prefixes
* Sticky ECMP is available in XR 6.3.2 for NCS5500 and ASR9000
* Auto Recovery is available in XR 6.5.1
