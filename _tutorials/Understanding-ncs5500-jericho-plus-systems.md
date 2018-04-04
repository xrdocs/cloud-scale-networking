---
published: true
date: '2018-03-30 22:50 +0200'
title: Understanding NCS5500 Jericho+ Systems and their scalability
author: Nicolas Fevrier
excerpt: Introduction to the NCS5500 Jericho+ Systems and their Scalability
tags:
  - iosxr
  - xr
  - ncs5500
  - jericho+
  - j+
position: top
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E06 Introduction of the Jericho+ based platforms and impact on the scale

### Previously on "Understanding NCS5500 Resources"

In previous posts, we presented:
- the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/)  
- we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/)
- we covered [how IPv6 prefixes are stored in the same databases](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-07-understanding-ncs5500-resources-s01e03/).
- [we demonstrated in a video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-12-30-full-internet-view-on-base-ncs-5500-systems-s01e04/) how we can handle a full IPv4 and IPv6 Internet view on "Base" systems and line cards (i.e. without external TCAM, only using the LEM and LPM internal to the forwarding ASIC)
- finally in the fifth post, [we demonstrated in another video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2018-01-25-s01e05-large-routing-tables-on-scale-ncs-5500-systems/) the scale we can reach on Jericho-based systems with an external TCAM

In this episode we will introduce and study a second generation of line cards and systems based on an evolution of the Forwarding ASIC.

### Jericho+

This Forwarding ASIC from Broadcom re-uses all of the principles of the Jericho generation, simply extending some scales:
- the **bandwidth capabilities** and consequently, the interfaces count: we can now accomodate 9x 100G interfaces line rate per ASIC
- the **forwarding capability** , extending it to 835MPPS (the performance is the same with lookups in internal databases or external TCAM)
- some memories like **LPM** (for certain models) and **EEDB**. Also we will use a **new generation eTCAM** (significantly larger).

"Certain models"? Yes, J+ exists in different flavors. Some are re-using the same LEM/LPM scale than Jericho and some others have a larger LPM memory (qualified for 1M to 1.3M instead of 256K to 350+K IPv4 entries).

![J-J+-FE3600.jpg]({{site.baseurl}}/images/J-J+-FE3600.jpg) 

Both Jericho and Jericho+ can be used with current Fabric Cards (FE3600). Some restrictions may apply for the 16-slot chassis, please contact your local account team to discuss the specifics.

### New systems using this J+ ASIC

In the modular chassis:

In March 2018, we have a single line card using Jericho+: the 36x100G-A-SE (more LC are coming in the summer).

![36x100G-SE.jpg]({{site.baseurl}}/images/36x100G-SE.jpg)

The line card is timing capable (note: an RP-E is necessary to use these timing features) and only exists in scale version (with eTCAM). The supported scale in current release is 4M+ IPv4 entries. It does not include MACsec chipset. It's also the first line card supporting break-out cable 4x 25G.

Internally, the line is composed of 4 Jericho+ (each one handling 9 ports QSFP). As shown in this diagram, each Jericho+ Forwarding ASIC is connected to the fabric cards via 8x 25G SERDES instead of 6 in the case of Jericho-based line cards.

![36x100G-SE-1.jpg]({{site.baseurl}}/images/36x100G-SE-1.jpg)

We are also extending the fixed-form factor portfolio with 3 new 1RU options:

- NCS55A1-36H-S
- NCS55A1-36H-SE-S
- NCS55A1-24H

Let's get started with the 36 ports options.

These standalone systems are MACsec + timing capable and are available in base (**NCS55A1-36H-S**) and scale versions (**NCS55A1-36H-SE-S**). Both have the same port density.

The base version shows the same route scale than a Jericho systems without external TCAM while the scale version uses a new generation eTCAM extending the scale to 4M IPv4 routes (potentially much more in the future).

![NCS55A1-36H-SE-S.jpg]({{site.baseurl}}/images/NCS55A1-36H-SE-S.jpg)

Internally, the system is composed of 4 Jericho+ ASICs (each one handling 9 ports QSFP) interconnected via an FE3600 chipset.

![NCS55A1-36H-SE-S-1.jpg]({{site.baseurl}}/images/NCS55A1-36H-SE-S-1.jpg)

The third router: **NCS55A1-24H**. 

It's a cost optimized, oversubscribbed, system that provides 24 ports QSFP. It is timing-capable but doesn't support MACsec.

![NCS55A1-24H.jpg]({{site.baseurl}}/images/NCS55A1-24H.jpg)

As shown in this diagram, the forwarding ASICs are connected back-to-back without using any fabric engine. Each ASIC handles 12 ports for a 900Gbps forwarding capability (hence the oversubscription).

![NCS55A1-24H-1.jpg]({{site.baseurl}}/images/NCS55A1-24H-1.jpg)

We will describe it in more details in the next sections but this system uses the largest version of Jericho+ ASICs. It doesn't use external TCAM but has a large LPM (1M to 1.3M prefixes instead of the 256K-350K we use on other systems in chassis or in the NCS55A1-36H-S).

### Let's talk about route scale

First, a quick reminder: the order of operation for route lookup in the NCS5500 family. It applies for both Jericho and Jericho+ systems.

![lookup-process.jpg]({{site.baseurl}}/images/lookup-process.jpg)

The prefixes are stored in LEM, LPM and when possible eTCAM.

### NCS55A1-36H-S Scale

On the NCS55A1-36H-S, the principles of prefixes storage are exactly the same than Jericho systems without eTCAM.

So it's possible to use two different modes:

- by default: the host mode

![36H-S-host.jpg]({{site.baseurl}}/images/36H-S-host.jpg)

- changed by configuration: the internet mode

![36H-S-internet.jpg]({{site.baseurl}}/images/36H-S-internet.jpg)

I invite you take a look at the [second](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/) and [third](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-07-understanding-ncs5500-resources-s01e03/) episode of this series. You will find detailed explanations and examples with real internet views.

### NCS55A1-36H-SE-S Scale

The NCS55A1-36H-SE-S is using the same Jericho+ ASIC but completed with a new generation and much larger external TCAM. In current release, it's certified for 4M IPv4 prefixes but the memory capabilities are significantly larger. We will decide in the future if it's necessary to increase the tested/validated scale.

Also, please note that the way we sort routes is different between 6.3.15 and 6.3.2. It's very likely we will simplify everything in the next major release during the summer (stay tuned).

![36H-SE-S-6315-632.jpg]({{site.baseurl}}/images/36H-SE-S-6315-632.jpg)

The uRPF does not affect the scale of this eTCAM (on the contrary of the first generation where it was necessary to disable the dual capacity feature, reducing the eTCAM to 1M entries). Also, the hybrid ACLs are using a different zone of the eTCAM memory and don't affect the overall scale.
{: .notice--info}

The NG eTCAM is algorithmic and the free memory displayed in the following show commands will depend on the current table distribution (in prefix length), so don't be surprised to see numbers below 4M. Also the max estimated values are currently showing 100% regardless the number. It will be fixed in  next releases.

<div class="highlighter-rouge">
<pre class="highlight">
<code>

RP/0/RP0/CPU0: 5508-6.3.2#sh route sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            9          0          0           2160
connected                        8          1          0           2160
static                           3          0          0           720
bgp 100                          1224649    0          0           293915760
dagr                             0          0          0           0
Total                            <mark>1224669</mark>    1          0           293920800

RP/0/RP0/CPU0:5508-6.3.2#sh route ipv6 sum

Route Source                     Routes     Backup     Deleted     Memory(bytes)
local                            6          0          0           1584
connected                        5          1          0           1584
connected l2tpv3_xconnect        0          0          0           0
bgp 100                          36488      0          0           9632832
Total                            36499      1          0           9636000

RP/0/RP0/CPU0:5508-6.3.2#sh dpa resource iproute loc 0/1/CPU0

"iproute" DPA Table (Id: 24, Scope: Global)
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
 /16      13572        /17      8005
 /18      14055        /19      25974
 /20      40443        /21      45082
 /22      83722        /23      71750
 /24      395144       /25      2085
 /26      3362         /27      5736
 /28      15909        /29      17377
 /30      42508        /31      112
 /32      435876
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: <mark>1224717</mark>         1224717         1224717         1224717
                 Create Requests
                           Total: 1224771         1224771         1224771         1224771
                         Success: 1224771         1224771         1224771         1224771
                 Delete Requests
                           Total: 54              54              54              54
                         Success: 54              54              54              54
                 Update Requests
                           Total: 287990          287990          287990          287990
                         Success: 287989          287989          287989          287989
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0

RP/0/RP0/CPU0:5508-6.3.2#sh dpa resource ip6route loc 0/1/CPU0

"ip6route" DPA Table (Id: 25, Scope: Global)
--------------------------------------------------
IPv6 Prefix len distribution
Prefix   Actual            Capacity    Prefix   Actual            Capacity
 /0       3                 0            /1       0                 0
 /2       0                 0            /3       0                 0
 /4       0                 0            /5       0                 0
 /6       0                 0            /7       0                 0
 /8       0                 0            /9       0                 0
 /10      3                 0            /11      0                 0
 /12      0                 0            /13      0                 0
 /14      0                 0            /15      0                 0
 /16      9                 0            /17      0                 0
 /18      0                 0            /19      0                 0
 /20      0                 0            /21      0                 0
 /22      0                 0            /23      0                 0
 /24      0                 0            /25      0                 0
 /26      0                 0            /27      0                 0
 /28      0                 0            /29      0                 0
 /30      0                 0            /31      0                 0
 /32      0                 0            /33      0                 0
 /34      0                 0            /35      0                 0
 /36      0                 0            /37      0                 0
 /38      0                 0            /39      0                 0
 /40      5184              0            /41      0                 0
 /42      0                 0            /43      0                 0
 /44      0                 0            /45      0                 0
 /46      0                 0            /47      0                 0
 /48      31304             0            /49      0                 0
 /50      0                 0            /51      0                 0
 /52      0                 0            /53      0                 0
 /54      0                 0            /55      0                 0
 /56      0                 0            /57      0                 0
 /58      0                 0            /59      0                 0
 /60      0                 0            /61      0                 0
 /62      0                 0            /63      0                 0
 /64      5                 0            /65      0                 0
 /66      0                 0            /67      0                 0
 /68      0                 0            /69      0                 0
 /70      0                 0            /71      0                 0
 /72      0                 0            /73      0                 0
 /74      0                 0            /75      0                 0
 /76      0                 0            /77      0                 0
 /78      0                 0            /79      0                 0
 /80      0                 0            /81      0                 0
 /82      0                 0            /83      0                 0
 /84      0                 0            /85      0                 0
 /86      0                 0            /87      0                 0
 /88      0                 0            /89      0                 0
 /90      0                 0            /91      0                 0
 /92      0                 0            /93      0                 0
 /94      0                 0            /95      0                 0
 /96      0                 0            /97      0                 0
 /98      0                 0            /99      0                 0
 /100     0                 0            /101     0                 0
 /102     0                 0            /103     0                 0
 /104     3                 0            /105     0                 0
 /106     0                 0            /107     0                 0
 /108     0                 0            /109     0                 0
 /110     0                 0            /111     0                 0
 /112     0                 0            /113     0                 0
 /114     0                 0            /115     0                 0
 /116     0                 0            /117     0                 0
 /118     0                 0            /119     0                 0
 /120     0                 0            /121     0                 0
 /122     0                 0            /123     0                 0
 /124     0                 0            /125     0                 0
 /126     0                 0            /127     0                 0
 /128     12                0
                          NPU ID: NPU-0           NPU-1           NPU-2           NPU-3
                          In Use: <mark>36523</mark>           36523           36523           36523
                 Create Requests
                           Total: 36523           36523           36523           36523
                         Success: 36523           36523           36523           36523
                 Delete Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                 Update Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                    EOD Requests
                           Total: 0               0               0               0
                         Success: 0               0               0               0
                          Errors
                     HW Failures: 0               0               0               0
                Resolve Failures: 0               0               0               0
                 No memory in DB: 0               0               0               0
                 Not found in DB: 0               0               0               0
                    Exists in DB: 0               0               0               0

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources lem loc 0/1/CPU0

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

Current Usage
    NPU-0
        Total In-Use                : <mark>435877</mark>   <mark>(55 %)</mark>
        iproute                     : 435876   (55 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 0        (0 %)
    NPU-1
        Total In-Use                : 435877   (55 %)
        iproute                     : 435876   (55 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 0        (0 %)
    NPU-2
        Total In-Use                : 435877   (55 %)
        iproute                     : 435876   (55 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 0        (0 %)
    NPU-3
        Total In-Use                : 435877   (55 %)
        iproute                     : 435876   (55 %)
        ip6route                    : 0        (0 %)
        mplslabel                   : 0        (0 %)

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources lpm loc 0/1/CPU0

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>0</mark>        <mark>(0 %)</mark>
        iproute                     : 0        (0 %)
        ip6route                    : 0        (0 %)
        ipmcroute                   : 0        (0 %)
        
-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources exttcamipv4 loc 0/1/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv4

OOR Information
    NPU-0
        Estimated Max Entries       : 788841
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Red
        OOR State Change Time       : 2018.Mar.27 08:01:30 PDT

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>788841</mark>   (100 %)
        iproute                     : 788841   (100 %)
        ipmcroute                   : 0        (0 %)

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources exttcamipv6short loc 0/1/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv6_short

OOR Information
    NPU-0
        Estimated Max Entries       : 73016
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>73016</mark>    (100 %)
        ip6route                    : 73016    (100 %)

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu resources exttcamipv6long loc 0/1/CPU0

HW Resource Information
    Name                            : ext_tcam_ipv6_long

OOR Information
    NPU-0
        Estimated Max Entries       : 60
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

-- SNIP --

Current Usage
    NPU-0
        Total In-Use                : <mark>60</mark>       (100 %)
        ip6route                    : 60       (100 %)

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#sh contr npu externaltcam loc 0/1/CPU0

External TCAM Resource Information
=============================================================
NPU  Bank   Entry  Owner       Free     Per-DB  DB   DB
     Id     Size               Entries  Entry   ID   Name
=============================================================
0    0      80b    FLP         334744   <mark>788832</mark>  0    IPv4 UC
0    1      80b    FLP         0        0       1    IPv4 RPF
0    2      160b   FLP         218130   <mark>36505</mark>   3    IPv6 UC
0    3      160b   FLP         0        0       4    IPv6 RPF
0    4      80b    FLP         4096     0       81   INGRESS_IPV4_SRC_IP_EXT
0    5      80b    FLP         4096     0       82   INGRESS_IPV4_DST_IP_EXT
0    6      160b   FLP         4096     0       83   INGRESS_IPV6_SRC_IP_EXT
0    7      160b   FLP         4096     0       84   INGRESS_IPV6_DST_IP_EXT
0    8      80b    FLP         4096     0       85   INGRESS_IP_SRC_PORT_EXT
0    9      80b    FLP         4096     0       86   INGRESS_IPV6_SRC_PORT_EXT

-- SNIP --

RP/0/RP0/CPU0:5508-6.3.2#
</code>
</pre>
</div>

### NCS55A1-24H Scale

The NCS55A1-24H is very different from the other NCS5500 routers because it uses a pair of Jericho+ with large LPM. So it occupies a particular place between the non-eTCAM and the eTCAM systems.

This large LPM is algorithmic, so even if it's marketed for 1M IPv4 entries, it can fit much more depending on the prefix distribution:

<div class="highlighter-rouge">
<pre class="highlight">
<code>
HW Resource Information
    Name                            : lem

OOR Information
    NPU-0
        Estimated Max Entries       : 786432
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green

HW Resource Information
    Name                            : lpm

OOR Information
    NPU-0
        Estimated Max Entries       : <mark>1384333</mark>
        Red Threshold               : 95
        Yellow Threshold            : 80
        OOR State                   : Green
</code>
</pre>
</div>

Like the other non-eTCAM systems, we can use two different configurations: the host-optimized mode (default) and the internet-optimized mode.

![24H-host-internet.jpg]({{site.baseurl}}/images/24H-host-internet.jpg)

Let's take a full internet table made of 655487 v4 and 42852 v6 real routes
and check how it fits in this system.

With the host optimized mode:

![24h-host.jpg]({{site.baseurl}}/images/24h-host.jpg)

And with the internet optimized mode:

![24h-internet.jpg]({{site.baseurl}}/images/24h-internet.jpg)

### Conclusion

Three different options with the Jericho+ systems: J+ with Jericho-scale, J+ with large LPM, J+ with new generation eTCAM. They are used in one new line card offering very high route scalability (4M+ routes), and in three new 1RU systems.