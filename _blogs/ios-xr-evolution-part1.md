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


## Process Distribution Across Available Compute

For scalability, the router cluster processes' load needs to be distributed across the available compute. The IOS XR uses two modes of distribution.

- Localization
- Load distribution

The first distribution model uses **localization**, which performs _processing_ and _storage_ closer to the resource. With this model, a database specific to a node is located on that node. The storage/data part is already addressed in the previous section. Similarly, processes are placed on a node where they have greater interaction with the resource. For example, Address Resolution Protocol (ARP), interface manager (IM), Bidirectional Failure Detection (BFD), adjacency manager, and Forwarding Information Base (FIB) manager are located on the line cards and are responsible only for managing resources and tables on that line card. This enables IOS XR to achieve faster processing and greater scalability.

The second distribution model uses **load distribution**, in which additional route processors (RPs) are added to the system and processes are distributed across different RPs. Routing protocols, management entities, and system processes are examples of processes that can be distributed using this model.


![]({{site.baseurl}}/images/dev-corner/xr_ev/8_distribution.png){: .align-center}

## High Performance Messaging Infrastructure

So far, we talked about _scalable data partitioning_ and _intelligent process placement_. The next key piece is the messaging infrastructure that lets these processes communicate inter-node and intra-node.

Messaging is at the core of many architectures including IOS XR and is a difficult problem. Connecting two pieces is fine but connecting hundreds and thousands is a different ball game. If we look at a modular router chassis with 12,16,18 etc., slots and tens/hundreds of processes per slot, we can see that we are already looking at the problem of messaging among hundreds/thousands of end points. These numbers go up further in multi-chassis clusters. There are a variety of applications (routing protocols, RIBs, FIBs, ACL/QoS etc., features, platform applications, interface managers, manageability agents etc.) and their communication requirements vary drastically w.r.t. scale, reliability, performance etc. 

For example, the following picture shows several dimensions which have been analyzed extensively across various applications while developing and evolving the IOS XR's messaging infrastructure and some resulting design patterns:

![]({{site.baseurl}}/images/dev-corner/xr_ev/9_requirements.png){: .align-center}

Rest of this section looks at the following key design patterns that IOS XR has pioneered.

### Asynchronous IO

Synchronous programming is arguably easier to reason about and implement. Synchronous communication requires that the sender and the receiver be coupled together during the message exchange. Since the receiver may not be ready to receive a message when it is sent, synchronous communication requires that the sender be blocked until the message is processed by the receiver, and it is some sort of acknowledgement or reply from the receiver that allows the sender to be unblocked. Thus, with synchronous IPC, the sender and receiver are essentially in lockstep for the duration of the message exchange. 

With asynchronous communication the sender and receiver are uncoupled - the sender does not know when the receiver processed a message that was sent. The async communication results in better IPC throughput, less latency and better use of allocated scheduler time slice resulting in less thrashing of processes involved in IPC and better use of multi-core environment. Asynchronous IPC allows an application to function with a minimal number of threads.  

![]({{site.baseurl}}/images/dev-corner/xr_ev/10_async.png){: .align-center}

We in IOS XR have realized very early on (circa 2000) that to make the system scalable, asynchronous communication is a key construct in the system. The IOS XR's _Group Communications_, which is covered later, kicked off this journey right off the bat for the inter-node IO. Subsequently, due to extensive scalability analysis and experience gained during the CRS multi-chassis development, the Async IO became a key construct for a lot of intra-node communication as well. We found that applications which run with fewer threads have less effect on scheduler latencies, in general, and consequently are less of a burden on the overall system. With asynchronous IPC, the same sender's thread can have multiple outstanding messages at a time - no need for a thread per message being sent as is the case with synchronous IPC.

Hence IOS-XR is designed as a distributed system with loose coupling among various IOS XR instances across nodes. Different IOS XR nodes run independently; there are no master-slave relationships among the different cards running IOS XR code. The statement of loose coupling among the IOS XR nodes is further stated that the different instances of IOS XR can only learn of each other’s state and share data through asynchronous message passing. The asynchronous IO pattern is supported across point-to-point and point-to-multipoint connections.

One can see the Async IO being a key pattern for scalability these days. For example, when Node.js was launched (circa 2009) it became popular overnight. It was based on one strong fundamental that any time we make an out-of-process call, we should not be wasting the hardware resources and instead free up the thread to do other things.  ZeroMQ, one of the successful messaging architectures launched around 2007, prides itself in its background Async IO module, how it uses lock-free algorithms for its queues, how it scales to any number of cores, how CPU quantum is efficiently utilized etc. 

### Group communications

In a modular router, there are many applications that implicitly have a requirement to communicate in a group. Typically, these are processes on different nodes in a router that need to communicate to share data and/or to synchronize with each other. IOS XR is one of the first NOSs that has pioneered this Group Communications concept (depicted in the following picture) with the following characteristics:

- One-to-many reliable group communication
- Connectionless, asynchronous semantics
- Efficient resource/data location through reliable multicast

![]({{site.baseurl}}/images/dev-corner/xr_ev/11_group-comm.png){: .align-center}

Because of these characteristics, IOS XR Group Services offers a very scalable communication method when multiple nodes are involved. Reliable, multicast and asynchronous are the key words worth noting above. If we take the reliability away from the Group Communications, it is akin to the Pub-Sub pattern where a message is directed at many subscribers and publishers and subscribers are completely decoupled.

Pub-sub is aimed at scalability where the publisher pushes data without really worrying about who the subscribers are, when they come online/subscribe to receive messages, whether they are able to receive/keep up with the messages, whether they have crashed/joined late etc. The subscribers in Group Communication connect to the multicast group on the network to which the publisher sends the messages. But a simple pub-sub pattern without reliability built into it doesn't cut it for NOS applications' needs. Hence, IOS XR Group Communications stack also provides several advanced features like the following:

- communicating with only a subset of members in a group 
- various reliability semantics (covered below)
- give the sender more detailed information about the reception of a message: how many nodes it was sent to, how many nodes received it (acknowledged its reception), how many members it was delivered to, and which nodes failed to respond to the message (if any).
- open group communication - groups which allow non-members to send messages to all members in the group.
- flow control
- member notification support for a wide range of notifications (membership change, dropped messages, node in the group is down etc.) to aid application paradigm.



The following picture depicts open, closed and membership subset groups.

![]({{site.baseurl}}/images/dev-corner/xr_ev/12_subset.png){: .align-center}

In terms of drawing parallels with external software, ZeroMQ supports _Group Messaging_ with some of the above features. Many other message brokers today support the Group Communication pattern. Also in IOS XR Group Communications, messages are queued on the subscriber, which attempts to avoid the problem of slow subscribers. ZeroMQ also supports this pattern.

#### Message guarantee semantics


IOS XR Group communication supports _fire-and-forget_, at least one, most and all reliability semantics with respect to the number of far end entities from which the producer needs to receive acknowledgments.



![]({{site.baseurl}}/images/dev-corner/xr_ev/13_ack.png){: .align-center}



One can find similar semantics in Kafka and other messaging systems. For example, Kafka supports the following:

- No acknowledgement, fire and forget. Acks=0.
- Leader has persisted the message. Acks=1
- Leader and all In Sync Replicas have persisted the message. Acks=All

Similarly, RabbitMQ also has control over number of unacknowledged messages in flight.

### Messaging protocols with pluggable transports

The messaging protocols in IOS XR have been developed with the required abstractions at the south bound layer so that they can be transported over various layers. For example, they can be transported over Ethernet, various switch fabric, Linux TCP/IP stack, IOS XR TCP/IP stack etc. These protocols can also support multiple transport layers simultaneously. For example, Group Communications in CRS platforms is supported over Ethernet and switch fabric simultaneously. This is akin to ZeroMQ supporting multiple transports (inproc, ipc, tcp, pgm) to support threads in one process, processes in one box, processes in one network and multicast group based communication.


![]({{site.baseurl}}/images/dev-corner/xr_ev/14_messaging.png){: .align-center}

## Data Distribution and Access Design Patterns 

After scalable data partitioning, _intelligent process placement_ to work with that data, high performance messaging infrastructure to enable communication among the processes, we now explore the next important aspect -  the _data distribution_ and _access patterns_ that applications running in an XR system should use.

Note that data distribution and access is inherently different from the messaging infrastructure/IPC mechanism – the messaging infrastructure provides a means of communication, it does not define the approach as to how to present data to other parts of the system that need the data, or how an application locates its resources that it needs for its operation. These issues are fundamental to the applications that run in IOS XR and are reflected in the implicit decisions an application is built upon. 

Understanding inherent data characteristics of router cluster is essential in designing and optimizing the NOS properly. There are detailed studies to understand data access pattens for various file systems. Similar studies are being done to understand data patterns in cloud environment today.  

IOS XR is designed taking into account the data distribution and access patterns in a small single CPU IOS XR router to a large distributed router built from multiple chassis. Based on these insights, the IOS XR NOS data access and distribution characteristics can be categorized as follows:

![]({{site.baseurl}}/images/dev-corner/xr_ev/15_data.png){: .align-center}

### Data Distribution and Access Characteristics

#### Data Distribution Characteristics

Data distribution has four sub characteristics - size of the data, number of consumers, liveness of producers/consumers and tracking of producer.

For example, FIB routes is an example of a large amount of data that is going to be consumed by a large number of nodes in a distributed routing cluster. Operational data is an example of a large set of data that is going to be consumed by a few management agents. Throughput and latency are two performance metrics that matter in data distribution.

![]({{site.baseurl}}/images/dev-corner/xr_ev/16_rib.png){: .align-center}

![]({{site.baseurl}}/images/dev-corner/xr_ev/17_oper.png){: .align-center}

Many of the data sharing applications would want to know the state of the producer(s), i.e. if the producer is active or has gone down so that they can take appropriate action. Again, a good example of this is the interaction between a routing protocol and RIB. If the protocol has populated routes to RIB, RIB should be notified when the protocol goes down so that it can stale the routes, and in the event that the protocol does not come back, it would want to delete those routes and promote backup routes, if any.




![]({{site.baseurl}}/images/dev-corner/xr_ev/18_rib.png){: .align-center}


Many of the applications would also want to track the producers of the data for various purposes. Going back to the RIB example, the routing protocols populating routes to RIB are the producers and RIB is the consumer. RIB explicitly tracks each such producer so that when all the producers are done with their routes, RIB declares routing convergence. This is an important state for the operation of the router cluster.

In some cases, the data producer needs to be aware of individual consumers. In most cases, the producer needs to know only about the aggregate set of consumers, but in some cases, more intimate knowledge of the consumer is needed. For example, consider the case of RIB downloading routes to various FIBs across the routing cluster. Typically, this is done by putting all the consumers in a group and multicasting to them. If one consumer restarts, the producer should start a separate download session to that consumer so that it is aware of all the updates thus far while continuing to update the remaining consumers.  The producer cannot suspend downloads to the other consumers because network events may happen at the same time and delaying downloads results in an unacceptable network convergence.  Once the restarting consumer catches up with the rest of the group, the producer has to merge the consumer back with the larger multicast group.


#### Data Access Characteristics

On the data access front, if we look at breadth of data access, most data is accessed only by a few entities in the router cluster, and that there is a very limited set of data that is accessed very broadly. One fine example for this is BGP’s private internal RIB structures. This data is referenced solely by BGP itself and the manageability agents.  It is extremely large, and in many cases, is the majority of the data in the router cluster.  BGP uses this data for computing routes and then injects specific paths into the RIB.  Feature (ACL, QoS etc.) data is another example of data with small breadth of access in terms of number clients. By contrast, there are other data items, such as the interface database, system configuration etc., that are referenced by almost every other process in the router cluster. 

Similarly, frequency of access of data items is highly variable, with a few data items that are very frequently accessed and the vast majority that are rarely accessed. For example, BGP provides an excellent example of a great deal of data that is accessed infrequently.  Since BGP only makes incremental changes, once a path is in BGP’s internal RIB, and advertised to the RIB processes and any neighbors, that information may not be accessed again until the advertising BGP session fails.  This could easily be days or weeks.  On the high frequency side, the interface database can be accessed extremely frequently, such as every time an application needs to resolve an interface name into a data structure, access interface dependent statistics or configuration. Thus routing and configuration are a couple of examples of data that is accessed infrequently while statistics, interfaces are data that are accessed frequently.

For some data, consumer consumes the data obtained from the producer as is. In some cases, consumers need to transform the received data.

### Design Patterns

Based on the above examples, we can observe some usage patterns. There is small, frequently and broadly accessed data and large, infrequently and sparsely accessed data. There are large volumes of data that need to be moved to a lot of nodes and there are small volumes of data that need to be moved to a lot of nodes.

For some of this data distribution, throughput or latency or both matter. 

In some cases, once data moves to a location, it needs to stay there in a database for the subsequent accesses. In some cases, producer and consumer are completely decoupled, while in some other cases, they need to track each other. In some instances, data is consumed as is between producer and consumer and in some other instances the data transformation is required.

What is the best way to satisfy these requirements? The IOS XR, after a careful consideration of data access metrics like the breadth of access, data size, frequency of access, degree of producer/consumer decoupling, liveness, data transformation requirements, is built around the following two fundamental data distribution/access patterns:

#### Data-centric

![]({{site.baseurl}}/images/dev-corner/xr_ev/19_dc-broker.png){: .align-center}

In a _data-centric_ approach, IOS XR applications are structured more around the data that is read or written. The identity of the processes that provide or consume the data is hidden in the infrastructure. Data is located by its description and that acts as the rendezvous point for end applications. To that end, it mimics the publisher-subscriber model – more popularly known as “pub-sub”. Since processes that share data will not be connecting to each other to exchange data, this model is characterized by a looser coupling between processes exchanging data.

The time, space and synchronization decoupling is achieved in IOS XR systems via a broker between producers and consumers. 

IOS XR designed the following two data-centric infrastructures based on the needs:

- SysDB - primarily a database with pub-sub messaging support.
- ENS (Event Notification System) - A data centric pub-sub messaging network.

![]({{site.baseurl}}/images/dev-corner/xr_ev/20_ens.png){: .align-center}

The above categorization is very important to design a good NOS. For example, if we look at configuration data, it is essentially state that is relatively small, fairly static, with a large set of consumers and with almost no need of data transformation. Databases are great for storing such state and keep retrieving it. But when something changes in there, it is good to have "don't call us, we will call you" support. Thus SysDB provides messaging/notification support. SysDB, like Redis, is an advanced in-memory data structure server. Interestingly, like Redis, SysDB also acts separately as a publish/subscribe server.

ENS, on the other hand is a decentralized flat topic based pub-sub messaging infrastructure that actually moves data from one place to another with reliability semantics. It is useful for the distribution of data that is written by one process to one or more processes on different nodes (a push model).  ENS also works equally well in a pull model. Readers can come up and pull the data from the writer somewhere in the system. It is decentralized as it is a collection of brokers spread across all nodes.

If we compare between ENS and SysDB, ENS creates direct links between distributed nodes, whereas SysDB is a logically centralized node which must be written to then read from. In this respect, this is similar to the comparison between ZeroMQ and Redis respectively. ZeroMQ/ENS is primarily a messaging infrastructure where as Redis/SysDB is primarily a database. SysDB is covered in more detail in the following information section.

**SysDB** - Logically centralized, physically distributed, scalable, neutral, in-memory, highly available, model driven, pub-sub datastore 
{: .notice--primary}


That is a mouthful of a description for SysDB but each word in there is important and collectively they define the concept and the power of SysDB. Configuration and operational data is significant part of the router state mentioned above and the IOS XR's SysDB is an advanced datastore to hold this data in a NOS.

![]({{site.baseurl}}/images/dev-corner/xr_ev/21_sysdb.png){: .align-center}


- Logically centralized, physically distributed

The configuration and operational data is eventually consumed by various manageability agents interacting with the router on the northbound side. From operations point of view,  it is hugely important to present the manageability agents the configuration/operational data in a centralized fashion irrespective of the internal implementation.  But internally the datastore should be distributed to achieve the required scalability, reliability and responsiveness goals of large systems. IOS XR designed SysDB as a scalable _distributed data store_ that presents a _single logical view_ for the whole system's configuration and operational data. Data is partitioned according to the principles mentioned in the earlier section. Logically centralized/physically distributed is a concept that we see these days being used in many SDN Controllers as they try to present one logical global view of the network state while physically distributing the state across multiple nodes.

- Scalable

SysDB is scalable to large number of nodes, large sets of configuration, high volume operational data and to frequently changing operational data.

One can apply more than two million lines of configuration and IOS XR works without a hiccup! One can retrieve large amounts of operational data from SysDB easily. This scalability can be attributed to proper data partition and distribution of the required processing of the same across the available compute.

Due to carefully designed shared _state concurrency_ and _data/processing load distribution_ across the compute, many applications access the data in parallel yielding greater scalability.

- Neutral

SysDB stores data in a format that is independent of the data formats of the manageability agents that it interacts with in the northbound direction and application backends it interacts with in the southbound direction. This is important for decoupling SysDB producers and consumers.  All manageability agents work on the same data and any new manageability agent support can be easily added.

With SysDB applications do not have to track the data by its location, nor do they have to track the location of the data providers. Since the applications are unaware of the identity or location of data providers/consumers, they are unaffected if these parties crash or relocate.

- Model Driven

SysDB data is modeled and has evolved over the years to support native and Openconfig Yang models. For many years, XR has provided programmatic access to operational data. 

- In-memory

The SysDB data is stored in memory (RAM). This is critical for the required performance.

- Pub-sub

SysDB is a hierarchical topic based _publish-subscribe_ mechanism that decouples producers and consumers. It is designed to store fairly static data as well as dynamic, fast-changing  and/or high volume data. Subscribers are usually interested in particular events or event patterns, and not in all events. There are different ways of specifying the events of interest, like topic-based, and more sophisticated _content-based_. In the simplest form, a topic is nothing but a unique string name; every topic is an event service on its own at which publishers and subscribers meet. In SysDB, the topics are unique string tuples. Topic name tuples can also be searched using wildcards, which offer the possibility to subscribe and publish to several topics whose names match a given set of keywords, like an entire sub tree or a specific level in the hierarchy. The actual data format stored in the leaves of the namespace tree is opaque to SysDB.

- Highly Available

Active/standby HA is supported.


#### Process-centric

![]({{site.baseurl}}/images/dev-corner/xr_ev/22_proc-centric.png){: .align-center}

In the process-centric model, specific processes own the data, and any process interested in the data must contact the owning process to retrieve or update the data. Processes discover each other through a _distributed directory services_ and register with each other for their interest on data.

One main benefit of the process-centric approach is the ability of the consumer to efficiently get liveness information about the producer. In cases where that liveness impacts the usability of the data, this is essential. This is not possible in a true data-centric system, as the producer is kept anonymously behind the publishing mechanism.

Since the producer of the data is aware of the data semantics, efficient data structures that are well-suited for the data being produced are used in IOS XR to build efficient notification mechanisms. 

There are many use cases among router applications where process-centric model turns out to be the best choice and IOS XR makes the optimal use of this construct.

## High Availability Foundation

![]({{site.baseurl}}/images/dev-corner/xr_ev/23_ha.png){: .align-center}



IOS XR is a _modular system_ with separate and _upgradeable components_.

The fact that IOS XR services run outside of the kernel in separate address spaces means that the crash of an IOS XR process is isolated and does not crash the system and, in general, does not crash other processes. Thus, a measure of fault isolation is available and it can be exploited for purposes of service upgrade. Restartable processes are a major aspect of the high availability of an IOS XR system and all IOS XR processes are required to be restartable.

_Decoupled planes abstraction_, explained earlier, also helps in the high availability of the system.

For restart and redundancy support, in general, _checkpointing_ and _replication_ services for local as well as remote checkpointing/replication are available so that restarted applications can start up warm or even hot. Replication services that allow checkpointing to more than one replica at a time are also available.

IOS XR also supports many levels of _redundancy_ in the system, including switch-over of paired RPs, OIR scenarios and process-level and role redundancy.

IOS XR has _runtime monitoring_ of CPU and memory resources, so that applications that are using up too much of either of these system resources may be terminated or have their scheduling semantics modified to reduce their impact on the rest of that node.

IOS XR also supports NSF (Non-stop Forwarding), GR protocols and NSR (Non-stop Routing).

The overall coordination of a router cluster, comprised of either single chassis or multiple chassis, requires that there be communications within the cluster, some sense of the topology of the cluster, and some consistent decision making capabilities within the cluster.  To do this, the IOS XR has a top-level control protocol that is responsible for electing an overall leader of the cluster and computing the topology of the cluster. 

## Upgradeability Architecture

The modularity of IOS XR makes it possible to upgrade individual or smaller pieces of the software. That is, the same architectural features that accomplish fault isolation can also be exploited to achieve a finer-grain level of software upgradeability. The problem of software upgradeability starts at the smallest unit of software divisibility and goes to collections of these units. It is not at all a given with an IOS XR system that a software upgrade has much of an impact on the system. Much effort continually goes into reducing the impact of software upgrades, and it is the architecture of an IOS XR system, as well as the software organization, that enables this process.

Because much of an IOS XR process’s code may actually reside in _shared libraries_/DLLs, it is possible to replace/upgrade a portion of a process’s code by loading a new DLL. Because IOS XR processes are restartable, all that is required to pick up a new version of a DLL is to restart a process. Even some IOS XR processes do not restart, but simply unload and reload newer versions of DLLs.

The fundamental building block of IOS XR is the component. This is the minimum version-able unit of software. A component is potentially field replaceable. A component may contain a process’s executable file, DLLs for sharing code between processes, binary files for implementing a portion of the CLI, text files describing configuration rules, or other related data. A typical example of an IOS XR component is Border Gateway Protocol (BGP). The BGP is a component in the Cisco IOS XR system that includes the BGP process, the configuration and operational models and associated processes.

IOS XR software is organized into _packages_ at the highest level. Packages are a collection of components and contain meta-data that indicates the compatibility of the components in the package with other components. Packages contain features that can be installed/activated/deactivated at runtime on an individual node, a set of nodes, or across the entire IOS XR system.

## Conclusion

The blog started off by giving a brief introduction to IOS XR's rapid evolution over the years. It then explained the architecture strategy via five steps - _decoupled plane abstractions, state management, process distribution, high performance messaging infrastructure and data distribution/access patterns._ It also looked into the high availability and upgradeability aspects of IOS XR that span across the various architecture stages. This logical sequencing, followed by details on each, hopefully gave you a better perspective of the design thought process.

I hope this blog gave good insights into various powerful architectural patterns built into IOS XR and how the IOS XR has pioneered various popular scalable and highly available frameworks in the context of a NOS. More importantly, it tries to give a sense of what it takes to build a carrier grade NOS and how much of a key role the infrastructure plays in a well designed NOS. 

The IOS XR anticipated and baked in many architectural patterns that are considered trendy and cutting edge today - and the IOS XR evolution continues as networking scene keeps changing around it as it has been over the last several years.  IOS XR kept the foundations simple which made complex things possible over the years. These solid foundations and continuous evolutions made IOS XR the best NOS for decades to come. For IOS XR, the change is a process and the future is the destination.
