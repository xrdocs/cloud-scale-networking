---
published: true
date: '2018-01-25 01:04 +0100'
title: S01E05 Large Routing Tables on "Scale" NCS 5500 Systems
author: Nicolas Fevrier
excerpt: >-
  Post #5 on the NCS5500 Resources: very large routing table in eTCAM systems,
  illustrated in a YouTube Video
tags:
  - ncs 5500
  - ncs5500
  - demo
  - video
  - youtube
  - large scale
  - routing
  - lem
  - lpm
  - etcam
  - internet
position: top
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E05 Large Routing Tables on "Scale" NCS 5500 Systems

### Previously on "Understanding NCS5500 Resources"

In previous posts, we presented:
- the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/)  
- we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/)
- we covered [how IPv6 prefixes are stored in the same databases](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-07-understanding-ncs5500-resources-s01e03/).
- and finally [we demonstrated in a video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-12-30-full-internet-view-on-base-ncs-5500-systems-s01e04/) how we can handle a full IPv4 and IPv6 Internet view on base systems and line cards (i.e. without external TCAM, only using the LEM and LPM internal to the forwarding ASIC)

Today, we are pushing the limits further. We will take a much larger existing (ie. real) routing table (internet + a very large number of host routes), we will add a projection of the internet table to year 2025 and we will see how it can fit in a Jericho-based system with External TCAM

### The demo

In this YouTube video, we will:
- describe the line cards and systems using Jericho / Qumran-MX forwarding ASICs with external TCAM (identified with the "-SE" at the end of the Product ID)
- explain the different memories we can use to store the routes and what logic is used to decide where the different prefix types will go
- run the demo
	- first with a real very large routing table of 1.2M IPv4 and 64k IPv6 routes
    (the v4 table size comes from a full internet view, a large number of peering routes and 436k host routes)
    - then we will project ourself to 2025 and guesstimate how large the v4 and v6 public table will be
    - we will advertise these extra routes, see how the router absorbs them and how much free space we have left in the different memories

[![NCS5500 Route Scale with eTCAMDemo](https://img.youtube.com/vi/lVC3ppgi7ak/0.jpg)](https://www.youtube.com/watch?v=lVC3ppgi7ak){: .align-center}

[https://www.youtube.com/watch?v=lVC3ppgi7ak](https://www.youtube.com/watch?v=lVC3ppgi7ak)

### CLI outputs

We jump directly to the larger use-case: large internet table from 2025 with 436k IPv4 host routes.
Such large number of host routes can be caused by DDoS mitigation systems (the /32s being used to divert the traffic targeted to specific victims) or by L3 VMs migration between domains.

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-6.2.3#sh route sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            2          0          0           480
connected                        2          0          0           480
bgp 100                          1612272    0          0           386945280
dagr                             0          0          0           0
static                           0          0          0           0
Total                            <mark>1612276</mark>    0          0           386946240

RP/0/RP0/CPU0:TME-5508-6.2.3#sh route ipv6 un sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            2          0          0           528
connected                        2          0          0           528
connected l2tpv3_xconnect        0          0          0           0
bgp 100                          108243     0          0           28576152
static                           0          1          0           264
Total                            <mark>108247</mark>     1          0           28577472

RP/0/RP0/CPU0:TME-5508-6.2.3#sh dpa resources iproute loc 0/6/CPU0

"iproute" DPA Table (Id: 18, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution
Prefix   Actual            Capacity    Prefix   Actual            Capacity
 /0       1                 16           /1       0                 16
 /2       0                 16           /3       0                 16
 /4       1                 16           /5       0                 16
 /6       0                 16           /7       0                 16
 /8       16                16           /9       14                16
 /10      37                163          /11      107               327
 /12      288               654          /13      557               1309
 /14      1071              2620         /15      1909              4585
 /16      13572             33905        /17      8005              20309
 /18      23343             34068        /19      38018             69283
 /20      40443             101879       /21      45082             113343
 /22      148685            185575       /23      116728            165738
 /24      651486            884472       /25      2085              3439
 /26      3362              3603         /27      5736              2620
 /28      15909             2292         /29      17377             5568
 /30      42507             2292         /31      112               163
 /32      <mark>435847</mark>            16
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: <mark>1612298</mark>         1612298         1612298         1612298
                 Create Requests
                           Total: 2285630         2285630         2285630         2285630
                         Success: 2285630         2285630         2285630         2285630
                 Delete Requests
                           Total: 673332          673332          673332          673332
                         Success: 673332          673332          673332          673332
                 Update Requests
                           Total: 2680653         2680653         2680653         2680653
                         Success: 2680651         2680651         2680651         2680651
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0

RP/0/RP0/CPU0:TME-5508-6.2.3#sh dpa resources ip6route loc 0/6/CPU0

"ip6route" DPA Table (Id: 19, Scope: Global)
--------------------------------------------------
IPv6 Prefix len distribution
Prefix   Actual       Prefix   Actual
 /0       1            /1       0
 /2       0            /3       0
 /4       0            /5       0
 /6       0            /7       0
 /8       0            /9       0
 /10      1            /11      0
 /12      0            /13      0
 /14      0            /15      0
 /16      4            /17      0
 /18      0            /19      2
 /20      9            /21      3
 /22      4            /23      4
 /24      20           /25      6
 /26      15           /27      17
 /28      80           /29      2889
 /30      189          /31      132
 /32      10091        /33      684
 /34      454          /35      423
 /36      3738         /37      292
 /38      686          /39      165
 /40      7578         /41      219
 /42      417          /43      129
 /44      7899         /45      187
 /46      1498         /47      376
 /48      <mark>52338</mark>        /49      13
 /50      12           /51      4
 /52      16           /53      0
 /54      1            /55      8
 /56      11622        /57      16
 /58      2            /59      0
 /60      3            /61      0
 /62      1            /63      0
 /64      5152         /65      0
 /66      0            /67      0
 /68      0            /69      0
 /70      0            /71      0
 /72      0            /73      0
 /74      0            /75      0
 /76      0            /77      0
 /78      0            /79      0
 /80      0            /81      0
 /82      0            /83      0
 /84      0            /85      0
 /86      0            /87      0
 /88      0            /89      0
 /90      0            /91      0
 /92      0            /93      0
 /94      0            /95      0
 /96      1            /97      0
 /98      0            /99      0
 /100     0            /101     0
 /102     0            /103     0
 /104     1            /105     0
 /106     0            /107     0
 /108     0            /109     0
 /110     0            /111     0
 /112     0            /113     0
 /114     0            /115     4
 /116     0            /117     0
 /118     0            /119     0
 /120     0            /121     0
 /122     71           /123     0
 /124     15           /125     0
 /126     24           /127     18
 /128     731
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: <mark>108265</mark>          108265          108265          108265
                 Create Requests
                           Total: 171646          171646          171646          171646
                         Success: 171646          171646          171646          171646
                 Delete Requests
                           Total: 63381           63381           63381           63381
                         Success: 63381           63381           63381           63381
                 Update Requests
                           Total: 4               4               4               4
                         Success: 2               2               2               2
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0

RP/0/RP0/CPU0:TME-5508-6.2.3#sh contr npu resources lem loc 0/6/CPU0

HW Resource Information
    Name                            : <mark>lem</mark>

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-1
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-2
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-3
        Estimated Max Entries       : 786432
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : <mark>488186</mark>   (<mark>62 %</mark>)
        iproute                     : 435847   (55 %)
        ip6route                    : 52338    (7 %)
        mplslabel                   : 0        (0 %)
    NPU-1
        Total In-Use                : 488186   (62 %)
        iproute                     : 435847   (55 %)
        ip6route                    : 52338    (7 %)
        mplslabel                   : 0        (0 %)
    NPU-2
        Total In-Use                : 488186   (62 %)
        iproute                     : 435847   (55 %)
        ip6route                    : 52338    (7 %)
        mplslabel                   : 0        (0 %)
    NPU-3
        Total In-Use                : 488186   (62 %)
        iproute                     : 435847   (55 %)
        ip6route                    : 52338    (7 %)
        mplslabel                   : 0        (0 %)

RP/0/RP0/CPU0:TME-5508-6.2.3#sh contr npu resources lpm loc 0/6/CPU0

HW Resource Information
    Name                            : <mark>lpm</mark>

OOR Information
    NPU-0
        Estimated Max Entries       : 486043
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-1
        Estimated Max Entries       : 486043
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-2
        Estimated Max Entries       : 486043
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-3
        Estimated Max Entries       : 486043
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : <mark>55950</mark>    (<mark>12 %</mark>)
        iproute                     : 0        (0 %)
        ip6route                    : 55927    (12 %)
        ipmcroute                   : 0        (0 %)
    NPU-1
        Total In-Use                : 55950    (12 %)
        iproute                     : 0        (0 %)
        ip6route                    : 55927    (12 %)
        ipmcroute                   : 0        (0 %)
    NPU-2
        Total In-Use                : 55950    (12 %)
        iproute                     : 0        (0 %)
        ip6route                    : 55927    (12 %)
        ipmcroute                   : 0        (0 %)
    NPU-3
        Total In-Use                : 55950    (12 %)
        iproute                     : 0        (0 %)
        ip6route                    : 55927    (12 %)
        ipmcroute                   : 0        (0 %)

RP/0/RP0/CPU0:TME-5508-6.2.3#sh contr npu resources exttcamipv4 loc 0/6/CPU0

HW Resource Information
    Name                            : <mark>ext_tcam_ipv4</mark>

OOR Information
    NPU-0
        Estimated Max Entries       : 1638400
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-1
        Estimated Max Entries       : 1638400
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-2
        Estimated Max Entries       : 1638400
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green
    NPU-3
        Estimated Max Entries       : 1638400
        Red Threshold               : 95 %
        Yellow Threshold            : 80 %
        OOR State                   : Green

Current Usage
    NPU-0
        Total In-Use                : <mark>1176451</mark>  (<mark>72 %</mark>)
        iproute                     : 1176451  (72 %)
        ipmcroute                   : 0        (0 %)
    NPU-1
        Total In-Use                : 1176451  (72 %)
        iproute                     : 1176451  (72 %)
        ipmcroute                   : 0        (0 %)
    NPU-2
        Total In-Use                : 1176451  (72 %)
        iproute                     : 1176451  (72 %)
        ipmcroute                   : 0        (0 %)
    NPU-3
        Total In-Use                : 1176451  (72 %)
        iproute                     : 1176451  (72 %)
        ipmcroute                   : 0        (0 %)

RP/0/RP0/CPU0:TME-5508-6.2.3#
</code>
</pre>
</div>

The 2025 internet estimation [is described in the previous post](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-12-30-full-internet-view-on-base-ncs-5500-systems-s01e04/). As mentioned in this post and in the video, the method is certainly a matter of debate. Let's take it for what it is: an estimation.

In these use-cases with large public routing table and extreme amount of host routes, we are far from reaching the limits of the systems based on Jericho ASICs with External TCAMs.

We are using 62% of LEM, 12% of LPM and 72% of eTCAM.

All these counters can be streamed with telemetry. Example of visualization with Grafana:

![grafana.png]({{site.baseurl}}/images/grafana.png)


Important to understand that default carving in IOS XR 6.2.3 is allocating 20% for hybrid ACLs. This default behavior will change in releases 6.3.x onwards where we will allocate 100% of the space to IPv4 prefixes and it will be only when configuring hybrid ACLs that we will re-carve to allocation eTCAM space.

We can verify the current carving status with the following:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-6.2.3#sh contr npu externaltcam loc 0/6/CPU0

External TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0      80b    FLP         <mark>461952</mark>   <mark>1176448</mark>  15   <mark>IPV4 DC</mark>
0    1      80b    FLP         28672    0       76   INGRESS_IPV4_SRC_IP_EXT
0    2      80b    FLP         28672    0       77   INGRESS_IPV4_DST_IP_EXT
0    3      160b   FLP         26624    0       78   INGRESS_IPV6_SRC_IP_EXT
0    4      160b   FLP         26624    0       79   INGRESS_IPV6_DST_IP_EXT
0    5      80b    FLP         28672    0       80   INGRESS_IP_SRC_PORT_EXT
0    6      80b    FLP         28672    0       81   INGRESS_IPV6_SRC_PORT_EXT
1    0      80b    FLP         461952   1176448  15   IPV4 DC
1    1      80b    FLP         28672    0       76   INGRESS_IPV4_SRC_IP_EXT
1    2      80b    FLP         28672    0       77   INGRESS_IPV4_DST_IP_EXT
1    3      160b   FLP         26624    0       78   INGRESS_IPV6_SRC_IP_EXT
1    4      160b   FLP         26624    0       79   INGRESS_IPV6_DST_IP_EXT
1    5      80b    FLP         28672    0       80   INGRESS_IP_SRC_PORT_EXT
1    6      80b    FLP         28672    0       81   INGRESS_IPV6_SRC_PORT_EXT
2    0      80b    FLP         461952   1176448  15   IPV4 DC
2    1      80b    FLP         28672    0       76   INGRESS_IPV4_SRC_IP_EXT
2    2      80b    FLP         28672    0       77   INGRESS_IPV4_DST_IP_EXT
2    3      160b   FLP         26624    0       78   INGRESS_IPV6_SRC_IP_EXT
2    4      160b   FLP         26624    0       79   INGRESS_IPV6_DST_IP_EXT
2    5      80b    FLP         28672    0       80   INGRESS_IP_SRC_PORT_EXT
2    6      80b    FLP         28672    0       81   INGRESS_IPV6_SRC_PORT_EXT
3    0      80b    FLP         461952   1176448  15   IPV4 DC
3    1      80b    FLP         28672    0       76   INGRESS_IPV4_SRC_IP_EXT
3    2      80b    FLP         28672    0       77   INGRESS_IPV4_DST_IP_EXT
3    3      160b   FLP         26624    0       78   INGRESS_IPV6_SRC_IP_EXT
3    4      160b   FLP         26624    0       79   INGRESS_IPV6_DST_IP_EXT
3    5      80b    FLP         28672    0       80   INGRESS_IP_SRC_PORT_EXT
3    6      80b    FLP         28672    0       81   INGRESS_IPV6_SRC_PORT_EXT
RP/0/RP0/CPU0:TME-5508-6.2.3#
</code>
</pre>
</div>

We have still 461952 routes left in the external TCAM.

In a follow up post, we will detail the mechanisms available to mix base and scale line cards in the same chassis, and how your network needs to be designed for such requirements.
