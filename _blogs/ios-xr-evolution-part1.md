---
published: true
date: '2018-12-03 10:23 -0800'
title: IOS XR Evolution - Part 1
position: hidden
excerpt: >-
  In this blog we will discuss IOS XR’s software architecture, that is typically
  invisible, and describe IOS XR's architecture from its foundations up.
author: Jag Tangirala
tags:
  - iosxr
  - cisco
---

{% include toc %}
{% include base_path %}


## Introduction
This blog is the first in a series about IOS XR’s software architecture. I believe that many of today’s best-practice architecture/design patterns for a successful buildout of scalable network system, are rooted in the IOS XR's architecture.

In this series, I will explore the part of IOS XR that is typically invisible, and describe IOS XR's architecture from its foundations up. I go into why the architecture looks like it does, introduce key design patterns, and draw occasional parallels with other scalable and high-performance software.                                                                                                                                                                                               
## IOS XR Evolution

When IOS XR started its journey several years ago, Google, Facebook, AWS etc., were not around. The world had not yet begun to massively scale!

Many of the massively scalable distributed system patterns that are perhaps more familiar to software developers these days were not that popular yet (mostly limited to text books or were part of a few closed commercial software implementations).

There weren't yet messaging infrastructures like ActiveMQ, RabbitMQ, ZeroMQ, Nanomsg, Kafka etc., in-memory databases like Redis, Memcached, Hazelcast etc., or distributed file systems (DFS) like HDFS, GlusterFS, CEPH etc.

Around this time, Cisco had embarked on building the next generation Network Operating System (NOS) with one clear vision -  building a highly scalable, reliable, available, modifiable, high performance NOS for the Service Provider (SP) space that caters all the way from low end single chassis systems to massive high end multi-chassis routers. Pulling this off required leveraging the deep networking knowledge base from the Cisco IOS operating system. In addition, to meet the rigorous SP requirements for this NOS, a slew of ground breaking infrastructure and distributed systems architecture/design patterns (depicted in the following picture) were brought into the system.

![]({{site.baseurl}}/images/dev-corner/xr_ev/1_image2018-9-14_9-48-40.png){: .align-center}

Before we get into the details of the above IOS XR architecture patterns in detail in the rest of the blog, it is worth quickly going through a summary of how the IOS XR has evolved over the years. There are several dimensions across which the IOS XR has demonstrated its modularity and adaptability capabilities with some major evolutions over the years. The following table summarizes these evolutions:

![]({{site.baseurl}}/images/dev-corner/xr_ev/2_table.png){: .align-center}


Each of the above is a significant evolution and demonstrates IOS XR's ability to evolve and take on new challenges ✅
{: .notice--success}


## The IOS XR Architecture Strategy

So how does one architect a carrier grade NOS?

The Cisco IOS XR Network Operating System (NOS) is developed by not changing the solution to an existing problem but rather asking a different question by looking at the customer's networking needs for decades to come.  The high level and heavily simplified IOS XR architecture strategy can be explained via the following six steps:

![]({{site.baseurl}}/images/dev-corner/xr_ev/3_strategy.png){: .align-center}

- Appropriate higher level abstractions capture the _essence_ of the system and they are key in driving the subsequent architecture/design patterns.
- Once abstractions are in place, IOS XR has focused on large state management in a router. (State is simply the condition or quality of an entity at an instant in time and it is usually represented by data in the system; hence, state and data are used interchangeably in this blog).
- Once the state generation and distribution patterns are modeled correctly, the next step is the design and placement of the processes that work with this state.
- But processes in a router interact with each other within a node (intra-node) and across the nodes (inter-node), often times moving significant amount of data with specific latency requirements. Hence the next important logical step in the architecture is designing a high-performance messaging infrastructure.
- The final step, once the messaging infrastructure is in place, is to understand different data access and distribution patterns of various applications over the messaging infrastructure and design those constructs.
- The high availability and upgradeability considerations span across all the stages.


In the rest of the blog, we dig into the internals of each of the IOS XR architecture strategy steps, and discuss principles and trade-offs in each step. On this journey, we will try to find useful ways of thinking about IOS XR NOS—not just how it is architected, but also why it is architected that way, and what **to** look for in a good NOS in general.


### Decoupled Planes Abstraction

IOS XR is a multi-process, distributed network operating system with tall order goals as mentioned earlier. In order to deliver on those goals, strong architectural abstractions are necessary. The IOS XR is architecturally divided into the following three planes:

- Management plane
- Control plane
- Data plane

These planes are a categorization of the traffic handled by a router and they provide an abstraction for the architecture of the router software. The planes abstraction helps in hiding a great deal of implementation detail behind a clean and nice facade. The management plane implements the external user interface used by operators to configure and query the system. The control plane is responsible for determining routes to use for traffic flows and generally how traffic should be forwarded. Control plane protocols (e.g. routing protocols) exchange information with other devices. Data plane directs traffic flows through the device. Forwarding is typically performed by hardware (but can be software) and data plane software is responsible for setting up hardware to perform forwarding.


![]({{site.baseurl}}/images/dev-corner/xr_ev/4_arch.png){: .align-center}

While the above three planes are perhaps more well known, there is a relatively less visible Infrastructure Plane behind the three planes, providing various key architecture/design patterns that are the subject of the rest of this blog.



![]({{site.baseurl}}/images/dev-corner/xr_ev/5_infra.png){: .align-center}

### Scalable, Available State/Data Management Patterns

A NOS in a router produces a lot of _state_. This router state is created by external inputs (_sourced state_) as well as internal code flow (_generated state_). 

Some examples of this state/data are configuration data, routing data, interfaces data, high-availability data, feature data (ACL, QoS etc.), statistics, protocol data, environmental data, platform data, operational data etc.  This data, depending on its type, has different _access patterns_ (most data is accessed only by a few entities in the cluster, and that there is a very limited set of data that is accessed very broadly), _frequency patterns_(a few data items that are going to be very frequently accessed and the vast majority that will be rarely accessed), data set sizes (large routing tables, huge operational data, small configuration etc.) and so on. This influences the data _partitioning/placing_ as well as _data distribution_ and access mechanisms by different entities in the router cluster (discussed a bit later in this blog).

In order to make the overall system scalable and highly available, the following architecture patterns are built into the IOS XR after taking into account the state/data attributes mentioned above:

![]({{site.baseurl}}/images/dev-corner/xr_ev/6_pieces.png){: .align-center}


#### Distributed state partitioning

It's available across the available compute nodes (route processors, line card processors, external compute processors etc., across the route cluster) 

1. the sourced state is partitioned across available compute as required. 
2. the generated state is kept on the source node as much as possible.
3. data is distributed in such a way as to minimize communication among nodes

For example, system databases specific to the line card, such as interface-related configurations, interface states, and so on, are stored on the line card. Run-time configuration flows to the node (route processor, line card etc.) where it is applicable.

#### Shared state concurrency

When two or more processes have some shared state between them, XR is designed for concurrent data access by multiple clients.  For example, configuration and operational data can be accessed by multiple internal/external clients simultaneously.

#### Caching mechanisms

Carefully designed state caching mechanisms at appropriate nodes in the cluster.

#### State replication and consistency mechanisms

**Replication** is the process of synchronizing several copies of the same state located at different nodes in the router cluster and is used to increase the availability of data and to speed query evaluation. This is a fundamental requirement for IOS XR and provides a generic replication and consistency management scheme for multiple copies of an opaque data set. It supports and scales well for multi-chassis systems providing an asynchronous 1:N replication method that spans one or more chassis.

IOS XR also studied various **consistency** models in the context of distributed router cluster and designed its infrastructure so that applications can choose the required consistency model based on their needs. This is akin to Amazon's Dynamo providing various consistency models and letting application choose the required consistency model while taking into account its availability, performance etc., metrics.

IOS XR also provides **distributed virtual file system** built on top of its replication framework. The motivation for this distributed file system lies in the high availability requirements of IOS XR router clusters. Nodes can go down and come back online at any time. Applications on these nodes need to access files in a location-independent manner. Designating special nodes as file servers means the entire system is affected if the special nodes go down. This infrastructure is designed as a decentralized as all replicas are designed as connectionless, anonymous peers.

#### User space resident IOS XR state

This dimension deals with distribution of state within a node between user and kernel spaces. Since IOS XR has started its journey with QNX micro kernel, the IOS XR's state resides completely outside the kernel. IOS XR has maintained this pattern as it moved from QNX to Linux as its base operating system. 

This clean separation between user and kernel space enables several advantages like better stability of the system, lot more freedom to bring in, modify and optimize user space code and increased modularity and independent upgradeability of kernel/user space code modules. For example, the IOS XR TCP/IP stack is completely home grown and resides in the user space. It is designed while keeping high end NOS in mind with a lot of optimizations/features. This stack plays a key role in IOS XR BGP and various other protocols/features' high performance numbers.

#### Microservices

Microservice is a hot buzzword in the industry today. There is no golden rule for the architecture in microservice. If we go by different architectures implemented in the industry, we can see everybody has their own flavor of microservice architecture. There is no perfect or certain definition of microservice. Rather microservices architecture can be summarized by certain characteristics or principles. Of course, when IOS XR was designed, there was no microservices buzzword, but since the IOS XR focused on modularity, loose coupling, high availability etc., many of the microservice characteristics are built into the IOS XR's architecture. For example, the following table below captures key microservice characteristics and the IOS XR's corresponding behavior against each item:

|                                                                                                                                                           Microservice Characteristic |                                                                                                                                                                                                                                                                                                                                                                                                IOS XR Characteristics |
|--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------:|
|                                                         Any component/service's failure should be in isolation. Failure of one service should not make the whole application go down. |                                                                                                                                                                                                                                                      The component isolation is provided via process abstraction. Processes support restartability and failure of a process doesn't affect the overall router health. |
|                                                                                                                        Components should talk to each other via predefined protocols. |                                                                                                                                                                                                                                                                                                                    There are contracts among components and the communication is via a mutually agreed upon protocol. |
|                                                                                                    Each component in the system should be cohesive, independent, and self-deployable. | Individual components are bundled and delivered to customer as “software packages”. Each package provides a feature(s) which can be installed, activated or upgraded/downgraded at runtime by customers. Packages can be installed/activated on all or subset of nodes by customer at runtime. Different version of software packages can be activated on different nodes. Backup copies kept on the box to rollback. |
| Each component or microservice should have its own database, with whom only that service is interacting. No other component or service can fetch or modify the data in that database. | Each component's private data is in its own private memory space. It is not accessible to others.                                                                                                                                                                                                                                                                                                                     |
| Each component with a specific purpose.                                                                                                                                               | The components are well defined modules implementing protocols, forwarding, drivers, features, specific infrastructure etc.                                                                                                                                                                                                                                                                                           |
| Expose functionality as a service.                                                                                                                                                    | Each component, by design, exposes a public API that is used by other components.                                                                                                                                                                                                                                                                                                                                     |
| The system should have automated testing in place.                                                                                                                                    | Many automated testing suites exist and more are being developed.                                                                                                                                                                                                                                                                                                                                                     |
| Two or more loosely coupled components                                                                                                                                                | The smallest software building block is a component. A component can be a DLL or process. Each component has metadata which defines the characteristic of that component.                                                                                                                                                                                                                                             |


The following is another way to depict the configuration/operational state distribution in a IOS XR router cluster:




![]({{site.baseurl}}/images/dev-corner/xr_ev/7_partitioning.png){: .align-center}

Thus IOS XR is to be viewed as a fully distributed system with loose coupling between IOS XR instances or nodes. The different IOS XR nodes run independently and the data partitioning also helps with fault tolerance/high-availability and low latency.


### Process Distribution Across Available Compute

For scalability, the router cluster processes' load needs to be distributed across the available compute. The IOS XR uses two modes of distribution.

- Localization
- Load distribution

The first distribution model uses **localization**, which performs _processing_ and _storage_ closer to the resource. With this model, a database specific to a node is located on that node. The storage/data part is already addressed in the previous section. Similarly, processes are placed on a node where they have greater interaction with the resource. For example, Address Resolution Protocol (ARP), interface manager (IM), Bidirectional Failure Detection (BFD), adjacency manager, and Forwarding Information Base (FIB) manager are located on the line cards and are responsible only for managing resources and tables on that line card. This enables IOS XR to achieve faster processing and greater scalability.

The second distribution model uses **load distribution**, in which additional route processors (RPs) are added to the system and processes are distributed across different RPs. Routing protocols, management entities, and system processes are examples of processes that can be distributed using this model.


![]({{site.baseurl}}/images/dev-corner/xr_ev/8_distribution.png){: .align-center}


