---
published: true
date: '2018-07-09 14:55 +0200'
title: NCS5500 Things to Know / Q&A
author: Nicolas Fevrier
excerpt: Various questions and answers related to the NCS5500 platform and its software
position: top
---

{% include toc icon="table" title="NCS5500 Things to know" %} 

## NCS5500 (and some other XR platforms): Good to Know

We frequently see the same questions around the NCS5500 platform and its software. Individually, they probably don't deserved a dedicated article, so we create this specific post to relay them and bring some answers.

We will keep this one updated regularly.

**Revision**:
- 2018-07-09: First version
- 2018-07-11: Add link to software center, and fix the optics support URL
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
