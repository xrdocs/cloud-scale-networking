---
published: true
date: '2018-07-09 14:55 +0200'
title: NCS5500 Things to Know / Q&A
author: Nicolas Fevrier
excerpt: Various questions and answers related to the NCS5500 platform and its software
position: top
---
{% include toc icon="table" title="NCS5500 Things to know" %} 

You can find more content related to NCS5500 including routing memory management, VRF, URPF, ACLs, Netflow following this [link](https://xrdocs.io/cloud-scale-networking/tutorials/).

## NCS5500 (and some other XR platforms): Good to Know

We frequently see the same questions around the NCS5500 platform and its software. Individually, they probably don't deserved a dedicated article, so we create this specific post to relay them and bring some answers.

We will keep this one updated regularly.

**Revision**:
- 2018-07-09: First version
- 2018-07-11: Add link to software center, and fix the optics support URL
- 2018-07-26: Add section on PIDs + section on the scale per LC / systems
{: .notice--info}


### Understanding IOS XR Releases

First, you may have heard that IOS XR exists in two flavors, the 32-bit and the 64-bit versions.
In the case of an ASR9000, you may use one or the other but with care since some hardware may not be supported in one or the other. For the NCS5500, things are simpler: it only supports the 64-bit version. We introduced the platform with IOS XR 6.0.0 and in 64-bit only.

Note: platforms are not necessarily "participating" in all "trains". For instance, the 6.4.x is available for XRv9000 and ASR9000 but not for the NCS5500. NCS5500 portfolio can use 6.0.x, 6.1.x, 6.2.x and 6.3.x. It will be part of the 6.5.x with the upcoming release 6.5.1 for example.

_Image name format_

The format used in IOS XR releases naming is always following X.Y.Zz

X, Y, Z meaning are detailed in this [CCO web page](https://www.cisco.com/c/en/us/products/collateral/ios-nx-os-software/ios-xr-software/product_bulletin_c25-478699.html). For example 6.1.3, 6.1.31 and 6.2.25.

It's an unwritten rule that x.y.z5 are "business releases" that are bringing the same features than x.y.z but with many bug fixes. Therefore, 6.2.25 is coming after 6.2.2 but pre 6.2.3 (even if 3<25)
    
x.y.z1, x.y.z2 or x.y.z3 (like 6.1.31) are releases built specifically with a group of features and most of the time with a specific list of customers. They are supposed to be ran with a defined and scoped use-case. They are usually not available on the [Cisco Software Download web site](https://software.cisco.com/download/home/279017029).

_What means EFT, GA, etc..._

Different images are qualified with these acronyms:

- FCS stands for First Customer Shipment: it's the official image build published on the software center.
- EFT stands for Early Field Trial: it a precode provided to specific customers for an approved test case. It's an image built and tested specifically for an usage. It can only be given to customers via their account team and in agreement with the BU/Engineering.
- LA means Limited availability and is usually referring to images available for specific customers, images you will not find on the Cisco Software download website.
- GA stands for General Availability and refers to images available to all customers. Examples: 6.1.3, 6.2.25, 6.2.3, 6.3.2.
- EMR means Extended Maintenance Release and represents a specific images in a train which will be supported for longer time.


### Specific release per NCS5500 platform?

We have a single image for the NCS5500 entire family, regardless it's a fixed-form system or a chassis, could it be 4, 8 and 16 slots, and regardless of the forwarding ASIC (Qumran-MX, Jericho or Jericho+).

![software-path.png]({{site.baseurl}}/images/software-path.png)

Pick the link to 5508 image as indicated above, it's the link to all systems, not only for 5508.

### Understanding Product IDs

Each router, each part and each license has its own PID and it could be confusing.

First, the PID finish with an equal character ("=") represents a spare part.

Second, very ofter you will not find in the ordering or maintenance tool the same PID that the one you see in your router when using "show platform". It's because the tool are using the "bundle PID" made of the product itself and the RTU (right to use) license.


<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:TME-5508-1-6.3.2#sh platform
Node              Type                       State             Config state
--------------------------------------------------------------------------------
0/1/CPU0          <mark>NC55-36X100G-A-SE</mark>          IOS XR RUN        NSHUT
0/1/NPU0          Slice                      UP
0/1/NPU1          Slice                      UP
0/1/NPU2          Slice                      UP
0/1/NPU3          Slice                      UP
0/2/CPU0          <mark>NC55-36X100G</mark>               IOS XR RUN        NSHUT
0/2/NPU0          Slice                      UP
0/2/NPU1          Slice                      UP
0/2/NPU2          Slice                      UP
0/2/NPU3          Slice                      UP
0/2/NPU4          Slice                      UP
0/2/NPU5          Slice                      UP
0/6/CPU0          <mark>NC55-24H12F-SE</mark>             IOS XR RUN        NSHUT
0/6/NPU0          Slice                      UP
0/6/NPU1          Slice                      UP
0/6/NPU2          Slice                      UP
0/6/NPU3          Slice                      UP
0/7/CPU0          <mark>NC55-24X100G-SE</mark>            IOS XR RUN        NSHUT
0/7/NPU0          Slice                      UP
0/7/NPU1          Slice                      UP
0/7/NPU2          Slice                      UP
0/7/NPU3          Slice                      UP
0/RP0/CPU0        <mark>NC55-RP</mark>(Active)            IOS XR RUN        NSHUT
0/RP1/CPU0        <mark>NC55-RP</mark>(Standby)           IOS XR RUN        NSHUT
0/FC0             <mark>NC55-5508-FC</mark>               OPERATIONAL       NSHUT
0/FC1             <mark>NC55-5508-FC</mark>               OPERATIONAL       NSHUT
0/FC3             <mark>NC55-5508-FC</mark>               OPERATIONAL       NSHUT
0/FC5             <mark>NC55-5508-FC</mark>               OPERATIONAL       NSHUT
0/FT0             <mark>NC55-5508-FAN</mark>              OPERATIONAL       NSHUT
0/FT1             <mark>NC55-5508-FAN</mark>              OPERATIONAL       NSHUT
0/FT2             <mark>NC55-5508-FAN</mark>              OPERATIONAL       NSHUT
0/SC0             <mark>NC55-SC</mark>                    OPERATIONAL       NSHUT
0/SC1             <mark>NC55-SC</mark>                    OPERATIONAL       NSHUT
RP/0/RP0/CPU0:TME-5508-1-6.3.2#
</code>
</pre>
</div>

The line cards NC55-36X100G-A-SE is oftened seen in the CCO tool as NC55-36X100G-A-SB.  
NC55-36X100G-A-SB being a bundle made of:
- NC55-36H-SE-RTU (right to use license)
- NC55-36X100G-A-SE (line card)

Let's summarize the product IDs in this chart:

| PID | Description | Bundle? |
|:-----:|:-----:|:-----:|
| NCS-5508 | NCS5500 8 Slot Single Chassis |  |
| NCS-5516 | NCS5500 8 Slot Single Chassis |  |
| NCS-5504 | NCS5500 4 Slot Single Chassis |  |
| NC55-RP | NCS 5500 Route Processor |  |
| NC55-SC | NCS 5500 System Controller |  |
| NC55-PWR-3KW-AC | NCS 5500 AC 3KW Power Supply |  |
| NC55-PWR-3KW-DC | NCS 5500 DC 3KW Power Supply |  |
| NC55-5508-FAN | NCS 5508 Fan Tray |  |
| NC55-5508-FC | NCS 5508 Fabric Card |  |
| NC55-5516-FAN | NCS 5508 Fan Tray |  |
| NC55-5516-FC | NCS 5516 Fabric Card |  |
| NC55-5504-FAN | NCS 5504 Fan Tray |  |
| NC55-5504-FC | NCS 5504 Fabric Card |  |
| NC55-36X100G | NCS 5500 36X100GE BASE | NC55-36X100G-BA / NC55-36X100G-U-BA |
| NC55-36X100G-S | NCS 5500 36x100G MACsec Line Card | NC55-36X100G-BM / NC55-36X100G-U-BM |
| NC55-24X100G-SE | NCS 5500 24x100G Scaled Line Card | NC55-24X100G-SB |
| NC55-18H18F | NCS 5500 18X100G and 18X40GE Line Card | NC55-18H18F-BA |
| NC55-24H12F-SE | NCS 5500 24X100GE and 12X40GE Line Card | NC55-24H12F-SB |
| NC55-36X100G-A-SE | NCS 5500 36x100G-SE Line Card | NC55-36X100G-A-SB / NC55-36X100G-U-SB|
| NC55-6x200-DWDM-S | NCS 5500 6x200 DWDM MACsec Line Card | NC55-6X2H-DWDM-BM / NC55-2H-DWDM-BM |
| NC55-MOD-A-S | NCS 5500 12X10, 2X40 & 2XMPA Line Card Base, MACSec | NC55-MOD-A-BM |
| NC55-MPA-2TH-S | 2X200G CFP2 MPA |  |
| NC55-MPA-1TH2H-S | 1X200G CFP2 + 2X100G QSFP28 MPA |  |
| NC55-MPA-12T-S | 12X10G MPA |  |
| NC55-MPA-4H-S | 4X100G QSFP28 MPA |  |
| | |
| NCS-5501 | NCS 5501 Fixed 48x10G and 6x100G Chassis |  |
| NCS-5501-SE | NCS 5501 - 40x10G and 4x100G Scale Chassis |  | 
| NCS-1100W-ACFW | NCS 5500 AC 1100W Power Supply Port-S Intake / Front-to-back |  |
| NCS-1100W-ACRV | NCS 5500 AC 1100W Power Supply Port-S Exhaust/Back-to-Front |  |
| NCS-950W-DCFW | NCS 5500 DC 950W Power Supply Port-S Intake / Front-to-back |  |
| NCS-1100W-DCRV | NCS 5500 DC 1100W Power Supply Port-S Exhaust / Back-to-Front |  |
| NCS-1100W-HVFW | NCS 5500 1100W HVAC/HVDC Port-S Intake / Front-to-back |  |
| NCS-1100W-HVRV | NCS 5500 1100W HVAC/HVDC Port-S Exhaust / Back-to-Front |  |
| NCS-1RU-FAN-FW | NCS 5500 1RU Chassis Fan Tray Port-S Intake / Front-to-back |  |
| NCS-1RU-FAN-RV |NCS 5500 1RU Chassis Fan Tray Port-S Exhaust / Back-to-Front |  |
| NCS-5502 | NCS5502 Fixed 48x100G Chassis | |
| NCS-5502-SE | NCS5502 - 48x100G Scale Chassis | |
| NC55-2RU-FAN-RV | NCS 5500 Fan Tray 2RU Chassis Port-S Exhaust / Back-to-Front | |
| NC55-2RU-FAN-FW | NCS 5500 Fan Tray 2RU Chassis Port-S Intake / Front-to-back | |
| NC55-2KW-DCRV | NCS5500 DC 2KW Power Supply Port-S Exhaust/Back-to-Front | |
| NC55-2KW-DCFW |NCS 5500 DC 2KW Power Supply Port-S Intake / Front-to-back | |
| NC55-2KW-ACRV | NCS 5500 AC 2KW Power Supply Port-S Exhaust / Back-to-Front | |
| NC55-2KW-ACFW | NCS 5500 AC 2KW Power Supply Port-S Intake / Front-to-back | |
| NCS-5502-FLTR-FW | NCS 5502 Filter Port-side exhaust / Back-to-Front | |
| NCS-5502-FLTR-RV | NCS 5502 Filter Port-Side intake / Front-to-back | |
| NCS-55A1-24H | NCS55A1 Fixed 24x100G chassis bundle | NCS-55A1-24H-B |
| NC55-A1-FAN-FW | NCS 5500 Fan Tray 1RU Chassis Port-S Intake / Front-to-back Port-Side intake | |
| NC55-A1-FAN-RV | NCS 5500 Fan Tray 1RU Chassis Port-S Exhaust / Back-to-Front Port-side exhaust | |
| NCS-55A1-36H-S | NCS55A1 Fixed 36x100G Base chassis bundle | NCS-55A1-36H-B |
| NCS-55A2-MOD-S | NCS 55A2 Fixed 24X10G + 16X25G & MPA Chassis | |
| NC55-1200W-ACFW | NCS 5500 AC 1200W Power Supply Port-S Intake / Front-to-back | |
| NC55-930W-DCFW | NCS 5500 DC 930W Power Supply Port-S Intake / Front-to-back | |
| NC55-A2-FAN-FW | NCS 5500 Fan Tray 1RU Chassis Port-S Intake / Front-to-back Port-Side intake | |
| NC55-MPA-2TH-S  | 2X200G CFP2 MPA | |
| NC55-MPA-1TH2H-S | 1X200G CFP2 + 2X100G QSFP28 MPA | |
| NC55-MPA-12T-S | 12X10G MPA | |
| NC55-MPA-4H-S | 4X100G QSFP28 MPA | |
| NCS-55A2-MOD-HD-S | NCS 55A2 Fixed 24X10G + 16X25G & MPA Temp Hardened Chassis | |
| NC55-900W-ACFW-HD | NCS 5500 AC 900W Power Supply Port-S Intake / Front-to-back | |
| NC55-900W-DCFW-HD | NCS 5500 DC 900W Power Supply Port-S Intake / Front-to-back | |
| NC55-MPA-4H-HD-S | 4X100G QSFP28 Temp Hardened MPA | |

### Understanding NCS5500 slot numbering

Line cards count starts from 0, from top to bottom.  

![slot-numbering.png]({{site.baseurl}}/images/slot-numbering.png)


### Products, ASICs and route scale

With Qumran-MX, Jericho, Jericho+ with Jericho-scale, Jericho+ with large LPM, with or without eTCAM, it's not easy to remember which ASIC is used in the various LC and systems and what is the routing scale they can reach.

The following chart will help clarifying it:

| PID | ASIC type | # of ASICs | Route Scale |
|:-----:|:-----:|:-----:|:-----:|
| NC55-36X100G | Jericho w/o eTCAM | 6 | 786k in LEM + 256-350k in LPM |
| NC55-36X100G-S | Jericho w/o eTCAM | 6 | 786k in LEM + 256-350k in LPM |
| NC55-24X100G-SE | Jericho with eTCAM | 4 | 786k in LEM + 256-350k in LPM + 2M in eTCAM |
| NC55-18H18F | Jericho w/o eTCAM | 3 | 786k in LEM + 256-350k in LPM |
| NC55-24H12F-SE | Jericho with eTCAM | 4 | 786k in LEM + 256-350k in LPM + 2M in eTCAM |
| NC55-36X100G-A-SE | Jericho+ with NG eTCAM | 4 | 4M in eTCAM |
| NC55-6x200-DWDM-S | Jericho w/o eTCAM | 2 | 786k in LEM + 256-350k in LPM |
| NC55-MOD-A-S | Jericho+ w/o eTCAM | 1 | 786k in LEM + 256-350k in LPM |
| NC55-MOD-A-SE-S | Jericho+ w/ eTCAM | 1 | 786k in LEM + 256-350k in LPM + 4M in eTCAM |
| NCS-5501 | Q-MX w/o eTCAM | 1 | 786k in LEM + 256-350k in LPM |
| NCS-5501-SE | Jericho with eTCAM | 1 | 786k in LEM + 256-350k in LPM + 2M in eTCAM |
| NCS-5502 | Jericho w/o eTCAM | 8 | 786k in LEM + 256-350k in LPM |
| NCS-5502-SE | Jericho with eTCAM | 8 | 786k in LEM + 256-350k in LPM + 2M in eTCAM |
| NCS-55A1-24H | Jericho+ w/o eTCAM | 2 | 786k in LEM + 1M-1.3M in LPM |
| NCS-55A1-36H-S | Jericho+ w/o eTCAM | 4 | 786k in LEM + 256-350k in LPM |
| NCS-55A1-36H-SE-S | Jericho+ with NG eTCAM | 4 | 4M in eTCAM |
| NCS-55A2-MOD-S | Jericho+ w/o eTCAM | 1 | 786k in LEM + 256-350k in LPM |
| NCS-55A2-MOD-HD-S | Jericho+ w/o eTCAM | 1 | 786k in LEM + 256-350k in LPM |
| NCS-55A2-MOD-SE-S | Jericho+ w/o eTCAM | 1 | 786k in LEM + 256-350k in LPM + 4M in eTCAM |

Buffers:  
Each ASIC is associated by 4GB of GDDR5 memory (total is the multiplication of number of NPU by 4GB).  
The buffer size is not related to the -SE or non-SE aspect (it's not related to TCAM).  
For more details, refer to Lane's whitepaper:  
[https://xrdocs.io/cloud-scale-networking/blogs/2018-05-07-ncs-5500-buffering-architecture/](https://xrdocs.io/cloud-scale-networking/blogs/2018-05-07-ncs-5500-buffering-architecture/)


### Supported optics

The first link to bookmark is the following: Select the interface type and then NCS5500.

[https://www.cisco.com/c/en/us/support/interfaces-modules/transceiver-modules/products-device-support-tables-list.html](https://www.cisco.com/c/en/us/support/interfaces-modules/transceiver-modules/products-device-support-tables-list.html)

But what about the third party optics?

We are not preventing any optic to work on the system. No PID check similar to classic XR. Therefore, no special cli is needed to enable similar to classic XR or NxOS. But the behaviour is not guaranteed. Some optics may use non-standard features and we can not guarantee whether it will work as expected or not.
Check the [Third Party Components - Cisco Policy](https://www.cisco.com/c/en/us/products/prod_warranty09186a00800b5594.html) for the official company position on the matter.


### Ethernet only on grey interfaces

The NCS5500 is only ethernet capable ?

Indeed, no frame-relay or SDH/Sonet technologies supported on these ASICs

The NCS5500 is LAN-Phy only?

It's true with the exception of the modular systems and line cards (NCS55A2-MOD and NC55-MOD LCs). With these particular systems, we offer MPAs supporting WAN-Phy and OTN framing too.


### Breakout-cable

Do you support breakout cable options?

Yes, depending on the optic type, it's possible to configure 4x10G or 4x25G

<div class="highlighter-rouge">
<pre class="highlight">
<code>
RP/0/RP0/CPU0:NCS5500(config)#controller optics 0/0/0/2
RP/0/RP0/CPU0:NCS5500(config-Optics)# breakout 4x10
</code>
</pre>
</div>

It doesn't need a reload to be enabled.

Some optics like the SR4 can be natively broke out in 4x10G. Some others will need specific optics like the 4x10G LR. Check the pdf linked above for all options. 25G is only supported on the Jericho+ platforms.
{: .notice--info}

Do you support all the features on ports in breakout mode?

Yes, no restriction

Can you mix breakout ports and "normal" ports on the same NPU

Yes, no restriction in the combination of ports on a given NPU


### Interface identification

In QSFP based systems or line cards, we support multiple types of interfaces.

If you insert a QSFP28, the port will be seen as HundredGig 0/x/y/z
- the first value describe the rack-id for multi-chassis. Since we don't support MC on this platform, it will always be 0
- x being the slot number, starting from the top of the chassis with 0
- y being the position inside a modular line card (NC55-MOD***), if it's a non-MOD card, it will be 0
- z being the port position in the line card or the system
- [This xrdocs post](https://xrdocs.io/cloud-scale-networking/tutorials/2018-02-15-port-assignments-on-ncs5500-platforms/) detailed the ports for each platform

If you insert a QSFP+, the port will be seen as FortyGig 0/x/y/z by default
- same description for x, y and z
- depending on the optic type, it may be possible to configure the breakout option. In this case, the port will appear as TenGig 0/x/y/z/w. You notice a fifth tuple to describe the interface position. Check the configuration needed above in this article

If you insert a QSA optic (QSFP to SFP Adaptor), the port will appear as TenGig 0/x/y/z
