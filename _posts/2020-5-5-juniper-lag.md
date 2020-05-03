---
layout: post
title: Juniper LAGs
tags: [juniper, saltstack]
image: /img/juniper_logo.jpg
---

A LAG combines multiple physical links between two adjacent nodes and offers increases bandwidth, link efficiency and physical redundancy. Other protocols use the LAG as a single interface.

In order to make the other protocols rapidly detect failures, you can configure them with BFD. BFD supports OSPF, IS-IS, BGP, LDP, RSVP, static routes and more. 

In some networks, BFD is even configured on multiple protocols at the same time. But did you know that you can also protect a LAG using micro-BFD sessions? And that this protection also aids the higher-layer protocols?

### Micro-BFD on LAG interfaces


Protocols running across interfaces will react nearly instantaneously to an interface down event. So using BFD as a liveness detection protocol for the individual links between two systems will also ensure that higher layer-protocols (such as OSPF, LDP or BGP) will be able to respond quickly to the loss of connectivity.



### Configuring a LAG with BFD


In this example, we will configure a LAG between two MX routers:


Let's start out configuring the LAG. The LAG will use LACP and it will be configured with 4 child links. We do not want the LAG to remain up in case there is only 1 link left. To this end, we configure the following:

<pre style="font-size:12px">
set chassis aggregated-devices ethernet device-count 20

set interfaces ge-0/0/4 description vMX2
set interfaces ge-0/0/4 gigether-options 802.3ad ae0
set interfaces ge-0/0/5 description vMX2
set interfaces ge-0/0/5 gigether-options 802.3ad ae0
set interfaces ge-0/0/6 description vMX2
set interfaces ge-0/0/6 gigether-options 802.3ad ae0
set interfaces ge-0/0/7 description vMX2
set interfaces ge-0/0/7 gigether-options 802.3ad ae0

set interfaces ae0 description vMX-2
set interfaces ae0 aggregated-ether-options minimum-links 2
set interfaces ae0 aggregated-ether-options lacp active

set interfaces ae0 unit 0 family inet address 10.100.0.0/31
set interfaces ae0 unit 0 family inet6 address 2001:db8:1000::0/127
</pre>

The first line enables the system to configure a total of 20 LAGs. Without this configuration, the Juniper device will not bring up the LAG. After this, we assign 4 interfaces to LAG interface AE0. After this, we configure the AE0 interface configuration. Here we configure the <b>aggregated-ether-options</b> to use LACP and we instruct the system to bring the LAG down in case there are less then 2 links. 
Finally, we finish up configuring IP addresses on the AE interface.


Configuring this on both vMX-1 as well as vMX-2 is enough to bring up the LAG. But in this case, we also want to enable BFD between the two systems. The BFD sessions that protect the LAG are configured as part of the <b>aggregated-ether-options</b> under the interface configuration of the LAG.

First, we configure it on <b>vMX-1</b>:

<pre style="font-size:12px">
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 10.0.0.2
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 10.0.0.1

set interfaces ae0 unit 0 family inet address 10.100.0.0/31
set interfaces ae0 unit 0 family inet6 address 2001:db8:1000::0/127

set interfaces lo0 unit 0 family inet address 10.0.0.1/32
set interfaces lo0 unit 0 family inet6 address 2001:db8:1000::1/128
</pre>

Next on <b>vMX-2</b>:

<pre style="font-size:12px">
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 10.0.0.1
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 10.0.0.2

set interfaces lo0 unit 0 family inet address 10.0.0.2/32
set interfaces lo0 unit 0 family inet6 address 2001:db8:1000::2/128
</pre>




### The complete configuration

<b>vMX-1</b>:

<pre style="font-size:12px">
set chassis aggregated-devices ethernet device-count 20

set interfaces ge-0/0/4 description vMX2
set interfaces ge-0/0/4 gigether-options 802.3ad ae0
set interfaces ge-0/0/5 description vMX2
set interfaces ge-0/0/5 gigether-options 802.3ad ae0
set interfaces ge-0/0/6 description vMX2
set interfaces ge-0/0/6 gigether-options 802.3ad ae0
set interfaces ge-0/0/7 description vMX2
set interfaces ge-0/0/7 gigether-options 802.3ad ae0

set interfaces ae0 description vMX-2
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 10.0.0.2
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 10.0.0.1
set interfaces ae0 aggregated-ether-options minimum-links 2
set interfaces ae0 aggregated-ether-options lacp active

set interfaces ae0 unit 0 family inet address 10.100.0.0/31
set interfaces ae0 unit 0 family inet6 address 2001:db8:1000::0/127

set interfaces lo0 unit 0 family inet address 10.0.0.1/32
set interfaces lo0 unit 0 family inet6 address 2001:db8:1000::1/128
</pre>



<b>vMX-2</b>:

<pre style="font-size:12px">
set chassis aggregated-devices ethernet device-count 20

set interfaces ge-0/0/4 description vMX1
set interfaces ge-0/0/4 gigether-options 802.3ad ae0
set interfaces ge-0/0/5 description vMX1
set interfaces ge-0/0/5 gigether-options 802.3ad ae0
set interfaces ge-0/0/6 description vMX1
set interfaces ge-0/0/6 gigether-options 802.3ad ae0
set interfaces ge-0/0/7 description vMX1
set interfaces ge-0/0/7 gigether-options 802.3ad ae0

set interfaces ae0 description vMX-1
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 10.0.0.1
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 10.0.0.2
set interfaces ae0 aggregated-ether-options minimum-links 2
set interfaces ae0 aggregated-ether-options lacp active

set interfaces ae0 unit 0 family inet address 10.100.0.1/31
set interfaces ae0 unit 0 family inet6 address 2001:db8:1000::1/127

set interfaces lo0 unit 0 family inet address 10.0.0.2/32
set interfaces lo0 unit 0 family inet6 address 2001:db8:1000::2/128
</pre>











### Verifying the LAG and the BFD sessions








