---
published: false
date: '2018-07-09 14:55 +0200'
title: NCS5500 Things to Know
author: Nicolas Fevrier
excerpt: Various questions and answers related to the NCS5500 platform and its software
position: top
---

{% include toc icon="table" title="Understanding NCS5500 Resources" %} 

## NCS5500 (and some other XR platforms): Good to Know

In this post, we will try to answer various questions asked around the platform and the software. If none deserved a dedicated article, they all should be answered to clarify potential doubts.

We will keep this one updated regularly, don't hesitate to check if anything new in it could be of interest.

**Revision**:
- 2018-06-16: First version
{: .notice--info}


### Understanding IOS XR Releases

First, you may have heard that IOS XR exists in two flavors, the 32-bit and the 64-bit versions.
In the case of an ASR9000, you may use one or the other but with care since some hardware may not be supported with some kind of image. For the NCS5500, things are simpler since it only supports the 64-bit version. We introduced the platform with IOS XR 6.0.0 and in 64-bit only.

Note: NCS5500 is not "participating" in all "trains". For instance, the 6.4.x is available on XRv9000 and ASR9000 but not the NCS5500. We were part of 6.0.x, 6.1.x, 6.2.x and 6.3.x. We will be part of the 6.5.x with the upcoming release 6.5.1 for example.

- Format
The format used in IOS XR releases naming is always following X.Y.Zz form:
For example 6.1.3, 6.1.31 and 6.2.25.
	- It's an unwritten rule that x.y.z5 are "business releases" that are bringing the same features than x.y.z but with many bug fixes. Therefore, 6.2.25 is post 6.2.2 but pre 6.2.3 (even if 3<25)
	- x.y.z1, x.y.z2 or x.y.z3 (like 6.1.31) are releases built specifically with a group of features and most of the time with a specific list of customers using a defined and scoped use-case. They are usually not available on the [Cisco Software Download web site](https://software.cisco.com/download/home/279017029).

- What means EFT, GA, etc...
Different images are qualified with these acronyms.
	- EFT stands for Early Field Trial: it a precode provided to specific customers with an approved test case. It's an image built and tested specifically for an usage. It can only be given to customers via their account team and in agreement with the BU/Engineering.
- LA means Limited availability and is usually referring to images available for specific customers, images you will not find on the Cisco Software download website.
- GA stands for General Availability and refers to images available to all customers. Examples: 6.1.3, 6.2.25, 6.2.3, 6.3.2.
- EMR means Extended Maintenance Release and represents a specific images in a train which will be supported for longer time.



### Supported optics

- The first link to bookmark is the following:
https://www.cisco.com/c/dam/en/us/products/se/2017/9/Collateral/fretta-optics-compatibility.pdf

- But what about the third party optics?
We are not preventing any optic to work on the system. No PID check similar to classic XR. No special cli is needed to enable similar to classic XR or NxOS. But the behaviour is not known. Some optics may use non-standard features and we can not guarantee whether it will works as expected or not. 
Check the [Third Party Components - Cisco Policy](https://www.cisco.com/c/en/us/products/prod_warranty09186a00800b5594.html) for the official company position on the matter.


### Ethernet only on grey interfaces

- the NCS5500 is only ethernet capable
Indeed, no frame-relay or SDH/Sonet technologies support on these ASICs

- the NCS5500 is LAN-Phy only?
It's true with the exception of the modular systems and line cards (NCS55A2-MOD and NC55-MOD LCs).
With these particular systems, we offer MPAs supporting WAN-Phy and OTN framing too.


### Breakout-cable

- Do you support breakout cable options?
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

Note: Some optics like the SR4 can be natively broke out in 4x10G. Some others will need specific optics like the 4x10G LR. Check the pdf linked above for all options. 25G is only supported on the Jericho+ platforms.

- Do you support all the features on ports in breakout mode?
Yes, no restriction

- Can you mix breakout ports and "normal" ports on the same NPU
Yes, no restriction in the combination of ports on a given NPU
