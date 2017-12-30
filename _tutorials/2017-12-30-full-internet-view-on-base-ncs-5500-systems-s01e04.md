---
published: true
date: '2017-12-30 10:31 +0100'
title: Full Internet View on "Base" NCS 5500 Systems (S01E04)
author: Nicolas Fevrier
excerpt: >-
  Post #4 on the NCS5500 Resources with a YT video to illustrate full internet
  support on non-eTCAM systems
tags:
  - ncs5500
  - ncs 5500
  - demo
  - video
  - youtube
  - lem
  - lpm
  - routes
  - base
  - prefixes
position: hidden
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E04 Full Internet View on "Base" NCS 5500 Systems

### Previously on "Understanding NCS5500 Resources"

In previous posts, we introduced:
- the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/)  
- we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/)
- and [how IPv6 prefixes are stored in the same databases](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-07-understanding-ncs5500-resources-s01e03/).

Let's illustrate how we can handle a full IPv4 and IPv6 Internet view on base systems and line cards (i.e. without external TCAM, only using LEM and LPM).


### The demo

Following YouTube video will demonstrate we can handle multiple internet peers in the global routing table (15 full views on our demo) on base systems with the appropriate optimizatios.  
We will demo how we can monitor the important resources used to store routing information and finally we will project what could be the internet size if it follows 2017 trends and how long the systems will be handle the full v4/v6 views.

[![NCS5500 Route Scale Demo](https://img.youtube.com/vi/8Tq4nyP2wuA/0.jpg)]
(/https://www.youtube.com/watch?v=8Tq4nyP2wuA "NCS5500 Route Scale")


### Config and CLI

On this line card we have:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508#sh route sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
connected                        9          1          0           2400
local                            10         0          0           2400
static                           2          0          0           480
ospf 100                         0          0          0           0
bgp 100                          698440     0          0           167625600
isis 1                           0          0          0           0
dagr                             0          0          0           0
Total                            698461     1          0           167630880
  
RP/0/RP0/CPU0:NCS5508#sh route ipv6 un sum
  
Route Source                     Routes     Backup     Deleted     Memory(bytes)
connected                        5          0          0           1320
connected l2tpv3_xconnect        0          0          0           0
local                            5          0          0           1320
static                           0          0          0           0
bgp 100                          61527      0          0           16243128
Total                            61537      0          0           16245768

RP/0/RP0/CPU0:NCS5508#sh bgp sum

BGP router identifier 1.1.1.1, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 1000324
BGP main routing table version 1000324
BGP NSR Initial initsync version 624605 (Reached)
BGP NSR/ISSU Sync-Group versions 1000324/0
BGP scan interval 60 secs

BGP is operating in STANDALONE mode.

Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker         1000324    1000324    1000324    1000324     1000324     1000324

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
192.168.100.151   0  1000  665841  210331  1000324    0    0    6d23h     655487
192.168.100.152   0 45896  666232  210331  1000324    0    0    6d23h     656126
192.168.100.153   0  7018  664430  210331  1000324    0    0    6d23h     654330
192.168.100.154   0  1836  669052  210331  1000324    0    0    6d23h     658948
192.168.100.155   0 50300  646305  210331  1000324    0    0    6d23h     636208
192.168.100.156   0 50304  667405  210331  1000324    0    0    6d23h     657301
192.168.100.157   0 57381  667411  210331  1000324    0    0    6d23h     657307
192.168.100.158   0  4608  687673  276201  1000324    0    0    6d23h     677487
192.168.100.159   0  4777  676317  210331  1000324    0    0    6d23h     666213
192.168.100.160   0 37989  298774  210331  1000324    0    0    6d23h     288706
192.168.100.161   0  3549  664480  210331  1000324    0    0    6d23h     654376
192.168.100.163   0  8757  642587  210331  1000324    0    0    6d23h     632483
192.168.100.164   0  3257 1319462  360784  1000324    0    0    6d22h     654661
192.168.100.165   0  3258  664741  262418  1000324    0    0    6d22h     654661
192.168.100.166   0  4609  687524  163237  1000324    0    0    6d22h     677487

RP/0/RP0/CPU0:NCS5508#sh bgp ipv6 un sum

BGP router identifier 1.1.1.1, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0800000   RD version: 64373
BGP main routing table version 64373
BGP NSR Initial initsync version 61139 (Reached)
BGP NSR/ISSU Sync-Group versions 64373/0
BGP scan interval 60 secs
 
BGP is operating in STANDALONE mode.
 
 
Process       RcvTblVer   bRIB/RIB   LabelVer  ImportVer  SendTblVer  StandbyVer
Speaker           64373      64373      64373      64373       64373       64373
 
Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
2001:111::151     0   100   68925   10070    64373    0    0    6d23h      58860
2001:111::152     0   100   51103   10070    64373    0    0    6d23h      41038
2001:111::153     0   100   52945  109776    64373    0    0    6d23h      42880
2001:111::155     0   100   52754   10070    64373    0    0    6d23h      42689
2001:111::156     0   100   40481   10070    64373    0    0    6d23h      30417
2001:111::157     0   100   13997   10070    64373    0    0    6d23h       3932
2001:111::158     0   100   52944   10070    64373    0    0    6d23h      42879
2001:111::159     0   100   51465   10070    64373    0    0    6d23h      41400
2001:111::160     0   100   52737   10070    64373    0    0    6d23h      42672
2001:111::161     0   100   52918   10070    64373    0    0    6d23h      42853
2001:111::163     0   100   51428   10070    64373    0    0    6d23h      41364
 
RP/0/RP0/CPU0:NCS5508#sh bgp scale
 
VRF: default
 Neighbors Configured: 26     Established: 26
 
 Address-Family   Prefixes Paths    PathElem   Prefix     Path       PathElem
                                               Memory     Memory     Memory
  IPv4 Unicast    698440   9481781  698440     98.58MB    795.74MB   71.27MB
  IPv6 Unicast    61527    430984   61527      9.39MB     36.17MB    6.28MB
  ------------------------------------------------------------------------------
  Total           759967   9912765  759967     107.97MB   831.91MB   77.55MB
 
Total VRFs Configured: 0
 
RP/0/RP0/CPU0:NCS5508#sh bgp

BGP router identifier 1.1.1.1, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0000000   RD version: 1000324
BGP main routing table version 1000324
BGP NSR Initial initsync version 624605 (Reached)
BGP NSR/ISSU Sync-Group versions 1000324/0
BGP scan interval 60 secs
 
Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop            Metric LocPrf Weight Path
*> 1.0.4.0/22         192.168.100.151                        0 1000 1299 4826 38803 56203 i
*                     192.168.100.152                        0 45896 45896 4826 38803 56203 i
*                     192.168.100.153                        0 7018 7018 3257 4826 38803 56203 i
*                     192.168.100.154                        0 1836 1836 6939 4826 38803 56203 i
*                     192.168.100.155                        0 50300 50300 6939 4826 38803 56203 i
*                     192.168.100.156                        0 50304 50304 6939 4826 38803 56203 i
*                     192.168.100.157                        0 57381 57381 6939 4826 38803 56203 i
*                     192.168.100.158                        0 4608 4608 4826 38803 56203 i
*                     192.168.100.159                        0 4777 4777 2516 4713 2914 15412 4826 38803 56203 i
*                     192.168.100.160                        0 37989 37989 4844 4826 38803 56203 i
*                     192.168.100.161       2514             0 3549 3549 3356 1299 4826 38803 56203 i
*                     192.168.100.163                        0 8757 8758 6939 4826 38803 56203 i
*                     192.168.100.164         10             0 3257 3257 4826 38803 56203 i
*                     192.168.100.165         10             0 3258 3257 4826 38803 56203 i
*                     192.168.100.166                        0 4609 4608 4826 38803 56203 i
*> 1.0.4.0/24         192.168.100.151                        0 1000 1299 4826 38803 56203 i
*                     192.168.100.152                        0 45896 45896 4826 38803 56203 i
*                     192.168.100.153                        0 7018 7018 3257 4826 38803 56203 i
*                     192.168.100.154                        0 1836 1836 6939 4826 38803 56203 i
*                     192.168.100.155                        0 50300 50300 6939 4826 38803 56203 i
*                     192.168.100.156                        0 50304 50304 6939 4826 38803 56203 i
*                     192.168.100.157                        0 57381 57381 6939 4826 38803 56203 i
*                     192.168.100.158                        0 4608 4608 4826 38803 56203 i
*                     192.168.100.159                        0 4777 4777 2516 4713 2914 15412 4826 38803 56203 i
*                     192.168.100.161       2514             0 3549 3549 3356 1299 4826 38803 56203 i
*                     192.168.100.163                        0 8757 8758 6939 4826 38803 56203 i
*                     192.168.100.164         10             0 3257 3257 4826 38803 56203 i
*                     192.168.100.165         10             0 3258 3257 4826 38803 56203 i
*                     192.168.100.166                        0 4609 4608 4826 38803 56203 i
*> 1.0.5.0/24         192.168.100.151                        0 1000 1299 4826 38803 56203 i
*                     192.168.100.152                        0 45896 45896 4826 38803 56203 i
*                     192.168.100.153                        0 7018 7018 3257 4826 38803 56203 i
*                     192.168.100.154                        0 1836 1836 6939 4826 38803 56203 i
*                     192.168.100.155                        0 50300 50300 6939 4826 38803 56203 i
*                     192.168.100.156                        0 50304 50304 6939 4826 38803 56203 i
*                     192.168.100.157                        0 57381 57381 6939 4826 38803 56203 i
*                     192.168.100.158                        0 4608 4608 4826 38803 56203 i
*                     192.168.100.159                        0 4777 4777 2516 4713 2914 15412 4826 38803 56203 i
*                     192.168.100.161       2514             0 3549 3549 3356 1299 4826 38803 56203 i
*                     192.168.100.163                        0 8757 8758 6939 4826 38803 56203 i
*                     192.168.100.164         10             0 3257 3257 4826 38803 56203 i
*                     192.168.100.165         10             0 3258 3257 4826 38803 56203 i
*                     192.168.100.166                        0 4609 4608 4826 38803 56203 i
...
 
RP/0/RP0/CPU0:NCS5508#sh bgp ipv6 un

BGP router identifier 1.1.1.1, local AS number 100
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0xe0800000   RD version: 64373
BGP main routing table version 64373
BGP NSR Initial initsync version 61139 (Reached)
BGP NSR/ISSU Sync-Group versions 64373/0
BGP scan interval 60 secs
 
Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop            Metric LocPrf Weight Path
*>i::/0               2001:111::151            0    100      0 1299 i
* i2001::/32          2001:111::151            0    100      0 286 1103 1101 i
*>i                   2001:111::152                 100      0 7018 6939 i
* i                   2001:111::155                 100      0 57821 6939 i
* i                   2001:111::156                 100      0 8758 6939 i
* i                   2001:111::158                 100      0 50304 2603 1103 1101 i
* i                   2001:111::159                 100      0 22652 6939 i
* i                   2001:111::160                 100      0 6881 6939 i
* i                   2001:111::161                 100      0 50300 6939 i
* i                   2001:111::163                 100      0 1836 6939 i
*>i2001:4:112::/48    2001:111::151            0    100      0 6724 112 i
* i                   2001:111::152                 100      0 7018 6939 112 i
* i                   2001:111::153                 100      0 57381 42708 112 i
* i                   2001:111::155                 100      0 57821 6939 112 i
* i                   2001:111::156                 100      0 8758 9002 112 i
* i                   2001:111::158                 100      0 50304 2603 1103 112 i
* i                   2001:111::159                 100      0 22652 112 i
* i                   2001:111::160                 100      0 6881 112 i
* i                   2001:111::161                 100      0 50300 112 i
* i                   2001:111::163                 100      0 1836 112 i
*>i2001:5:5::/48      2001:111::151            0    100      0 174 48260 i
* i                   2001:111::152                 100      0 7018 174 48260 i
* i                   2001:111::153                 100      0 57381 42708 1299 174 48260 i
* i                   2001:111::155                 100      0 57821 12586 3257 174 48260 i
* i                   2001:111::158                 100      0 50304 1299 174 48260 i
* i                   2001:111::159                 100      0 22652 174 48260 i
* i                   2001:111::160                 100      0 6881 25512 174 48260 i
* i                   2001:111::161                 100      0 50300 174 48260 i
* i                   2001:111::163                 100      0 1836 174 48260 i
*>i2001:200::/32      2001:111::151            0    100      0 174 2914 2500 2500 i
* i                   2001:111::152                 100      0 7018 2914 2500 2500 i
* i                   2001:111::153                 100      0 57381 6939 2914 2500 2500 i
* i                   2001:111::155                 100      0 57821 6939 2914 2500 2500 i
* i                   2001:111::156                 100      0 8758 174 2914 2500 2500 i
* i                   2001:111::158                 100      0 50304 1299 2914 2500 2500 i
* i                   2001:111::159                 100      0 22652 3356 2914 2500 2500 i
* i                   2001:111::160                 100      0 6881 6939 2914 2500 2500 i
* i                   2001:111::161                 100      0 50300 2914 2500 2500 i
* i                   2001:111::163                 100      0 1836 174 2914 2500 2500 i
...
 
RP/0/RP0/CPU0:NCS5508#sh dpa resources iproute loc 0/2/CPU0

"iproute" DPA Table (Id: 21, Scope: Global)
--------------------------------------------------
IPv4 Prefix len distribution
Prefix   Actual       Prefix   Actual
 /0       3            /1       0
 /2       0            /3       0
 /4       3            /5       0
 /6       0            /7       0
 /8       16           /9       14
 /10      37           /11      107
 /12      288          /13      557
 /14      1071         /15      1909
 /16      13546        /17      7986
 /18      14000        /19      25861
 /20      40214        /21      44565
 /22      83070        /23      70129
 /24      388562       /25      1522
 /26      1338         /27      877
 /28      314          /29      510
 /30      782          /31      60
 /32      1170
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3           NPU-4           NPU-5
                          In Use: 698511          698511          698511          698511          698511          698511
                 Create Requests
                           Total: 698546          698546          698546          698546          698546          698546
                         Success: 698546          698546          698546          698546          698546          698546
                 Delete Requests
                           Total: 35              35              35              35              35              35
                         Success: 35              35              35              35              35              35
                 Update Requests
                           Total: 141991          141991          141991          141991          141991          141991
                         Success: 141991          141991          141991          141991          141991          141991
                    EOD Requests
                           Total: 0               0               0               0               0               0
                         Success: 0               0               0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0               0               0
                Resolve Failures: 0               0               0               0               0               0
                 No memory in DB: 0               0               0               0               0               0
                 Not found in DB: 0               0               0               0               0               0
                    Exists in DB: 0               0               0               0               0               0
 
RP/0/RP0/CPU0:NCS5508#sh dpa resources ip6route loc 0/2/CPU0
 
"ip6route" DPA Table (Id: 22, Scope: Global)
--------------------------------------------------
IPv6 Prefix len distribution
Prefix   Actual       Prefix   Actual
 /0       3            /1       0
 /2       0            /3       0
 /4       0            /5       0
 /6       0            /7       0
 /8       0            /9       0
 /10      3            /11      0
 /12      0            /13      0
 /14      0            /15      0
 /16      10           /17      0
 /18      0            /19      2
 /20      9            /21      3
 /22      4            /23      4
 /24      20           /25      6
 /26      15           /27      17
 /28      79           /29      1899
 /30      156          /31      128
 /32      9773         /33      640
 /34      435          /35      410
 /36      1618         /37      284
 /38      680          /39      161
 /40      2285         /41      212
 /42      397          /43      117
 /44      2283         /45      180
 /46      1465         /47      371
 /48      20154        /49      13
 /50      12           /51      4
 /52      16           /53      0
 /54      1            /55      8
 /56      11621        /57      16
 /58      2            /59      0
 /60      3            /61      0
 /62      1            /63      0
 /64      5177         /65      0
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
 /104     3            /105     0
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
 /128     735
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3           NPU-4           NPU-5
                          In Use: 61568           61568           61568           61568           61568           61568
                 Create Requests
                           Total: 61572           61572           61572           61572           61572           61572
                         Success: 61572           61572           61572           61572           61572           61572
                 Delete Requests
                           Total: 4               4               4               4               4               4
                         Success: 4               4               4               4               4               4
                 Update Requests
                           Total: 2629            2629            2629            2629            2629            2629
                         Success: 2628            2628            2628            2628            2628            2628
                    EOD Requests
                           Total: 0               0               0               0               0               0
                         Success: 0               0               0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0               0               0
                Resolve Failures: 0               0               0               0               0               0
                 No memory in DB: 0               0               0               0               0               0
                 Not found in DB: 0               0               0               0               0               0
                    Exists in DB: 0               0               0               0               0               0
 
RP/0/RP0/CPU0:NCS5508#sh contr npu resources lem location 0/2/CPU0

HW Resource Information
    Name                            : lem
 
OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
    NPU-1
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
    NPU-2
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
    NPU-3
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
    NPU-4
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
    NPU-5
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
 
Current Usage
    NPU-0
        Total In-Use                : 409888   (52 %)
        iproute                     : 389732   (50 %)
        ip6route                    : 20154    (3 %)
        mplslabel                   : 2        (0 %)
    NPU-1
        Total In-Use                : 409888   (52 %)
        iproute                     : 389732   (50 %)
        ip6route                    : 20154    (3 %)
        mplslabel                   : 2        (0 %)
    NPU-2
        Total In-Use                : 409888   (52 %)
        iproute                     : 389732   (50 %)
        ip6route                    : 20154    (3 %)
        mplslabel                   : 2        (0 %)
    NPU-3
        Total In-Use                : 409888   (52 %)
        iproute                     : 389732   (50 %)
        ip6route                    : 20154    (3 %)
        mplslabel                   : 2        (0 %)
    NPU-4
        Total In-Use                : 409888   (52 %)
        iproute                     : 389732   (50 %)
        ip6route                    : 20154    (3 %)
        mplslabel                   : 2        (0 %)
    NPU-5
        Total In-Use                : 409888   (52 %)
        iproute                     : 389732   (50 %)
        ip6route                    : 20154    (3 %)
        mplslabel                   : 2        (0 %)
 
RP/0/RP0/CPU0:NCS5508#sh contr npu resources lpm location 0/2/CPU0

HW Resource Information
    Name                            : lpm
 
OOR Information
    NPU-0
        Estimated Max Entries       : 430200
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2017.Dec.22 09:30:17 PST
    NPU-1
        Estimated Max Entries       : 430200
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2017.Dec.22 09:30:17 PST
    NPU-2
        Estimated Max Entries       : 430200
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2017.Dec.22 09:30:17 PST
    NPU-3
        Estimated Max Entries       : 430200
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2017.Dec.22 09:30:17 PST
    NPU-4
        Estimated Max Entries       : 430200
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2017.Dec.22 09:30:17 PST
    NPU-5
        Estimated Max Entries       : 430200
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Yellow
        OOR State Change Time       : 2017.Dec.22 09:30:17 PST
 
Current Usage
    NPU-0
        Total In-Use                : 350206   (81 %)
        iproute                     : 308779   (72 %)
        ip6route                    : 41414    (10 %)
        ipmcroute                   : 0        (0 %)
    NPU-1
        Total In-Use                : 350206   (81 %)
        iproute                     : 308779   (72 %)
        ip6route                    : 41414    (10 %)
        ipmcroute                   : 0        (0 %)
    NPU-2
        Total In-Use                : 350206   (81 %)
        iproute                     : 308779   (72 %)
        ip6route                    : 41414    (10 %)
        ipmcroute                   : 0        (0 %)
    NPU-3
        Total In-Use                : 350206   (81 %)
        iproute                     : 308779   (72 %)
        ip6route                    : 41414    (10 %)
        ipmcroute                   : 0        (0 %)
    NPU-4
        Total In-Use                : 350206   (81 %)
        iproute                     : 308779   (72 %)
        ip6route                    : 41414    (10 %)
        ipmcroute                   : 0        (0 %)
    NPU-5
        Total In-Use                : 350206   (81 %)
        iproute                     : 308779   (72 %)
        ip6route                    : 41414    (10 %)
        ipmcroute                   : 0        (0 %)
 
RP/0/RP0/CPU0:NCS5508#
</code>
</pre>
</div>

Configuration to enable the internet-optimized mode on non-eTCAM line cards:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5508#sh run | i hw-

Building configuration...
hw-module fib ipv4 scale internet-optimized
RP/0/RP0/CPU0:NCS5508#
</code>
</pre>
</div>

Configuration to enable streaming telemetry for the counters used for this video:

<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RP0/CPU0:NCS5508#sh run telemetry model-driven
    telemetry model-driven
     destination-group DGroup1
      vrf default
      address-family ipv4 192.168.100.141 port 5432
   encoding self-describing-gpb
   protocol tcp
  !
 !
 sensor-group fib
  sensor-path Cisco-IOS-XR-fib-common-oper:fib/nodes/node/protocols/protocol/vrfs/vrf/summary
 !
 sensor-group brcm
  sensor-path Cisco-IOS-XR-fretta-bcm-dpa-hw-resources-oper:dpa/stats/nodes/node/hw-resources-datas/hw-resources-data
 !
 sensor-group routing
  sensor-path Cisco-IOS-XR-ipv4-bgp-oper:bgp/instances/instance/instance-active/default-vrf/process-info
  sensor-path Cisco-IOS-XR-ip-rib-ipv4-oper:rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as/information
  sensor-path Cisco-IOS-XR-ip-rib-ipv6-oper:ipv6-rib/vrfs/vrf/afs/af/safs/saf/ip-rib-route-table-names/ip-rib-route-table-name/protocol/bgp/as/information
 !
 subscription fib
  sensor-group-id fib strict-timer
  sensor-group-id fib sample-interval 1000
  destination-id DGroup1
 !
 subscription brcm
  sensor-group-id brcm strict-timer
  sensor-group-id brcm sample-interval 1000
  destination-id DGroup1
 !
 subscription routing
  sensor-group-id routing strict-timer
  sensor-group-id routing sample-interval 1000
  destination-id DGroup1
 !
!
     
RP/0/RP0/CPU0:NCS5508#
</code>
</pre>
</div>

## "Wet-finger" Internet Growth Projection




