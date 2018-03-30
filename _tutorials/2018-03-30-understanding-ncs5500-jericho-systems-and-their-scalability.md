---
published: false
date: '2018-03-30 22:50 +0200'
title: Understanding NCS5500 Jericho+ Systems and their scalability
---
{% include toc icon="table" title="Understanding NCS5500 Resources" %}  

## S01E06 Introduction of the Jericho+ based platforms and impact on the scale

### Previously on "Understanding NCS5500 Resources"

In previous posts, we presented:
- the [different routers and line cards in NCS5500 portfolio](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-02-understanding-ncs5500-resources-s01e01/)  
- we explained [how IPv4 prefixes are sorted in LEM, LPM and eTCAM](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-03-understanding-ncs5500-resources-s01e02/)
- we covered [how IPv6 prefixes are stored in the same databases](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-08-07-understanding-ncs5500-resources-s01e03/).
- [we demonstrated in a video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2017-12-30-full-internet-view-on-base-ncs-5500-systems-s01e04/) how we can handle a full IPv4 and IPv6 Internet view on base systems and line cards (i.e. without external TCAM, only using the LEM and LPM internal to the forwarding ASIC)
- finally in the fifth post, [we demonstrated in another video](https://xrdocs.github.io/cloud-scale-networking/tutorials/2018-01-25-s01e05-large-routing-tables-on-scale-ncs-5500-systems/) the scale we can reach on Jericho-based systems with an external TCAM

In this episode we will introduce a second generation of line cards and systems based on an evolution of the Forwarding ASIC.

### Jericho+

This Forwarding ASIC re-uses most of the principles of the Jericho generation, simply extending some scales:
- the bandwidth capabilities and then, the interfaces count: we can now accomodate 9x 100G interfaces line rate
- the forwarding capability , extending it to 835MPPS (same performance with lookup in internal databases or external TCAM)
- some resources like LPM (for certain models) and EEDB. Also we will use a new generation eTCAM (significantly larger).
Certain models? Yes, J+ exists in different flavors: some are re-using the same LEM/LPM scale than Jericho and some others have a larger LPM memory (qualified for 1M to 1.3M instead of 256K to 350+K IPv4 entries).

