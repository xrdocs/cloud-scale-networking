---
published: true
date: '2018-03-08 21:33 +0200'
title: Introduction to P4 and P4Runtime
author: Praveen Bhagwatula
excerpt: >-
  P4 is a domain specific programming language for writing packet forwarding functions. It has several native language constructs such as IPv4 addresses, Interfaces, VLANs that allows for writing highly optimal code which is portable (subject to some constraints as discussed below) across different NPU architectures.
tags:
  - IOS-XR
  - Cisco
  - OCP
  - P4
  - P4runtime
  - OFA
  - API
position: top 
---

{% include toc %}
{% include base_path %}

# Introduction to P4
## A Brief History of Programmable NPUs
Programmable NPUs have existed in routing products for a long time. Almost all the routers developed for the Service Provide and Enterprise market segments have been built with Network Processor Units (NPU) that are highly programmable, so as to keep up with the pace of new feature and functional requirements that are a norm in this market segment.  On the other hand, Cloud Provider market has traditionally used fixed function ASICs to build the networks, driven primarily by high bandwidth, low to medium functional requirements.

Environments for programming these NPUs historically fall into one of the following two categories:

1) Low level programming language, with target specific instruction set and tools. While this environment allows for writing most optimal forwarding plane code, it is tied to a  specific NPU architecture and not readily portable to others. This is analogous to writing assembly language code specific to X86 or ARM processor families which is not portable from one to the other.

2) High level programming language, such as C or C++, with target specific compilers and tools. This allows for writing code that is easy to port across multiple NPUs, or at-least provides a common environment for developing packet processing functions across multiple NPU architectures. While compilers have come a long way to optimize high level language code to target architectures, lack of native language constructs will result in sub-optimal performance in several cases.

## What is P4?
[P4](https://p4.org/) is a [domain specific programming language](https://en.wikipedia.org/wiki/Domain-specific_language) for writing packet forwarding functions. It has several native language constructs such as IPv4 addresses, Interfaces, VLANs that allows for writing highly optimal code which is portable (subject to some constraints as discussed below) across different NPU architectures.

To illustrate this with an example, the following is a very simple packet forwarding flow. IP destination address of the incoming packet is looked up in a longest prefix match table. If there is a match, packet is forwarded to the outgoing interface from the result of the lookup. If there is no match, the packet is dropped.

![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/C6ABE927-E86D-4E77-867D-A25188CCA088.png?raw=true)

A program written in a high level language such as C or Python would involve defining data structures for the IPv4 header, LPM table, results of the lookup and actions performed.

A sample P4 program that achieves the same functionality is shown below.
As seen, the language provides native constructs for objects such as IPv4 header (_ipv4_), table type (_LPM_), actions (_drop, route_), making it simpler to program.

```
table routing {
key = { ipv4.dstAddr : lpm; }
actions = { drop; route; }
size : 2048;
}
control ingress() {
apply {
routing.apply();
}
}    
```

P4 compilers generate two sets of outputs for a P4 program, as shown in the figure below.
	* A set of APIs that can be used by the control plane functions to program the table entries in the NPU
	* Target specific executable that can be loaded onto the packet processing NPU engines.

![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/BDB4CAF6-982A-4E49-9922-76358702B3E2.png?raw=true)


## Some observations about P4
The fundamental premise of P4 is that it allows the packet forwarding behavior of a NPU to be fully programmable using a domain specific language and the associated tools that makes it highly performant, yet portable across a set of NPUs, thus striking the balance between performance and portability.

There is however, a subtlety that is often easy to miss when discussing programmable NPUs; and that is the underlying architecture of the NPU itself. While most programmable NPUs look the same and provide similar levels of flexibility in terms of their programmability, the design of the packet processing engines within these NPUs - the blocks that deal with processing the incoming and outgoing packets - can vary significantly. This is attributed to factors such as the number and type of pipeline stages, level of programmability of different pipeline stages, connectivity and access rates to various memories from each of these stages, packet recycling and queuing architectures , etc., - driven by the design trade-offs made by NPU designers.

These architectural differences among the NPUs lead to an interesting observation about the P4 programs that are written to run on these NPUs - that, while most of the programs written can be compiled to run on most programmable NPUs, the same program may not yield the most optimal outcome in terms of performance, scale and convergence on all the NPUs, despite the fact that the backend compiler can map the P4 program to a target architecture.

Secondly, analysis of the NPUs  on the market shows there are aspects of the packet forwarding functionality that need to be implemented outside of the core P4 functionality, using platform specific extensions and SDK driven configurations. This leads to an observation that not all parts of a common P4 program may be compiled to run on a target NPU.

Thirdly, as the adoption for P4 as the packet forwarding programming language of choice, there is a desire to expand the scope of P4 to include not only the programmable NPUs that can run a compiled P4 program, but also fixed function or semi-programmable/configurable ASICs.

## P4 Usage Models
The adoption of P4, and the desire to use it across fully programmable, semi-programmable and fixed function ASICs, is driving the industry towards two distinct, yet related usage models.

### Running P4 compiled code on the NPU engines
As per for original intent of P4, developers program the packet forwarding functionality using P4 language constructs. A front-end compiler processes the program and generates a set of APIs for managing the tables and counters that the program operates on. These APIs are then used by the control plane application interfacing with the NPU. A back-end compiler then generates the target specific machine code for the P4 program, which is loaded onto the NPU engine as an executable that processes the packets. This model applies to programmable NPUs.

### P4 as a behavioral specification language
A second use case is the use of P4 as a behavioral specification for packet forwarding behavior. Vendors and customers write P4 programs in a generic way - not necessarily optimized for a specific NPU architecture - to represent the desired behavior they expect out of the packet forwarding function on the router or switch. A front-end compiler processes this program, as in the above case, and generates the set of APIs to manage the tables and counters accessed by the program.

The P4 program may or may not be compiled to run on the target NPU. When it is not compiled to run on the target NPU, it is either because the NPU is not programmable, or because the P4 program, as written, cannot be compiled to optimally run on the NPU. Instead, the control plane application invokes to the generated APIs, and a manual adaptation of these APIs to the SDK/driver for the target NPU is provided by the NPU vendor.

Some in the industry are looking to use P4 in this model. This helps for an unambiguous documentation of requirements between the customers and vendors, and also allows an effective way of validating how well a customer implementation matches (or not) with these requirements.  ONF has recently announced project [Stratum](https://www.opennetworking.org/news-and-events/press-releases/stratum-onf_google-launches-major-new-open-source-sdn-switching-platform-with-support-from-google/)that proposes use of P4Runtime, along with gNMI and gNOI as a way to manage an SDN switch.

# P4Runtime
[P4Runtime](https://p4.org/p4-runtime/), a project under P4.org’s API working group, is an effort towards developing an infrastructure for applications to interface with APIs generated from a P4 program in a seamless fashion. It uses GRPC with GPB encoding between a P4Runtime Controller (running locally on the switch or remotely) and P4Runtime Agent (running locally on the switch). NPU specific adaptation of the P4Runtime Agent APIs provides the required glue layer to interface with the SDK or the NPU driver.


![](https://github.com/xrdocs/cloud-scale-networking/blob/gh-pages/images/913D19A4-6BEA-47C5-8CBC-A152256AEC7A.png?raw=true)


As seen in the figure above, there are several ways that a P4Runtime Agent API gets mapped to the underlying hardware:

1) Direct mapping of the P4Runtime API to the program running on the NPU. This is the model what is applicable when the P4 program is compiled to run on the NPU without any modifications.

2) Adaptation of P4Runtime API to a NPU SDK that manages a NPU programmed using a target specific P4 or non-P4 program.

3) Adaptation of P4Runtime API to a NPU SDK managing a fixed function or semi-programmable NPU.

By abstracting the communication between the Controller (which the application interfaces with), and the Agent (which interfaces with the packet processing engine) using well define, version controlled APIs,  P4Runtime provides a consistent environment for applications to interface with switches and routers that have packet processing engines ranging from fully programmable NPUs to fixed function ASICs.

In summary, P4 is evolving as a programming language and environment of choice for vendors and customer alike, for behavioral specification and implementation of packet forwarding functionality on routers and switches.  Cisco is making investments in P4, both as a data plane programming environment and integrating the P4Runtime environment with IOS-XR.
