---
layout: post
title: BFD protected LAG
tags: [ juniper ]
image: /img/juniper_logo.jpg
---


A LAG combines multiple physical links between two adjacent nodes together to establish a single (virtual) link. This offers increased bandwidth, link efficiency and physical redundancy.

In order to make protocols that run across the LAG rapidly detect failures, you can configure them with BFD. BFD supports OSPF, IS-IS, BGP, LDP, RSVP, static routes and more. 

In some networks, BFD is configured for multiple protocols at the same time. But did you know that you can also protect a LAG using BFD? And that this protection also helps the higher-layer protocols to respond faster?

### Micro-BFD on LAG interfaces


When BFD is used as a liveness detection protocol for a LAG, micro-BFD sessions will monitor the forwarding path of the links between the two systems. The BFD sessions that are protecting the member links are independent BFD sessions. There is one BFD session per link that is part of the LAG. When BFD detects a failure in the path of a link, the child of the LAG is brought down. This way, BFD can detect failures in the forwarding path of a child link and ensure that it is brought down swiftly.

The BFD protected LAG will be able to respond to failures as fast as the BFD timers you configure. This in turn ensures that higher layer-protocols (such as OSPF, LDP or BGP) will be able to respond quickly to the loss of connectivity. This is because protocols running across interfaces will react nearly instantaneously to an interface down event. Consider an OSPF neighbor relationship with a dead timer of 40 seconds. When the underlying interface that is used to sustain the OSPF session is brought down by BFD, the system does not have to wait for the dead timer to reach 0. As soon as the LAG is brought down, the OSPF session is removed and alternate routes (if any) are considered. 

The same thing goes for other protocols and this makes it a strategy that can work out well for all sorts of networks, be it an MPLS core or a clos fabric.


### Configuring BFD for a LAG on Juniper MX


{:refdef: style="text-align: center;"}
![Juniper BFD protected LAG on MX](/img/juniper_lag_bfd.png "Juniper BFD protected LAG")
{: refdef}

Let's start out configuring the LAG with its 4 child links. Additionally, let's make sure the LAG does not remain up in case there is only 1 link left. To this end, we configure the following:

<b>vMX-1</b>:

<pre style="font-size:12px">
set chassis aggregated-devices ethernet device-count 20

set interfaces ge-0/0/4 gigether-options 802.3ad ae0
set interfaces ge-0/0/5 gigether-options 802.3ad ae0
set interfaces ge-0/0/6 gigether-options 802.3ad ae0
set interfaces ge-0/0/7 gigether-options 802.3ad ae0

set interfaces ae0 description vMX-2
set interfaces ae0 aggregated-ether-options minimum-links 2
set interfaces ae0 aggregated-ether-options lacp active

set interfaces ae0 unit 0 family inet address 10.100.0.0/31
set interfaces ae0 unit 0 family inet6 address 2001:db8:1000::0/127
</pre>


<b>vMX-2</b>:

<pre style="font-size:12px">
set chassis aggregated-devices ethernet device-count 20

set interfaces ge-0/0/4 gigether-options 802.3ad ae0
set interfaces ge-0/0/5 gigether-options 802.3ad ae0
set interfaces ge-0/0/6 gigether-options 802.3ad ae0
set interfaces ge-0/0/7 gigether-options 802.3ad ae0

set interfaces ae0 description vMX-1
set interfaces ae0 aggregated-ether-options minimum-links 2
set interfaces ae0 aggregated-ether-options lacp active

set interfaces ae0 unit 0 family inet address 10.100.0.1/31
set interfaces ae0 unit 0 family inet6 address 2001:db8:1000::1/127
</pre>


The first line enables the system to configure a total of 20 LAGs. Without this configuration, the Juniper device will not bring up any LAG. After this, we assign 4 interfaces to LAG AE0 and configure the AE0 interface itself. Here we configure the <b>aggregated-ether-options</b> to use LACP and we instruct the system to bring the LAG down in case there are less than 2 links. Finally, we finish up configuring IP addresses on the AE interface.

Next up is the BFD sessions that protect the LAG. Those are configured as part of the <b>aggregated-ether-options</b> under the interface configuration of the LAG. We only need to specify the BFD session characteristics once in the AE configuration. This will make the Juniper device attempt to establish BFD sessions on every active child link that is participating in the LAG.

<b>vMX-1</b>:

<pre style="font-size:12px">
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 10.0.0.2
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 10.0.0.1

set interfaces lo0 unit 0 family inet address 10.0.0.1/32
</pre>

<b>vMX-2</b>:

<pre style="font-size:12px">
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 10.0.0.1
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 10.0.0.2

set interfaces lo0 unit 0 family inet address 10.0.0.2/32
</pre>

Alternatively, we can also choose to create the BFD sessions using IPv6 addresses. In that case, we could configure something like this:

<b>vMX-1</b>:

<pre style="font-size:12px">
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 2001:db8:1000::2
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 2001:db8:1000::1

set interfaces lo0 unit 0 family inet6 address 2001:db8:1000::1/128
</pre>

<b>vMX-2</b>:

<pre style="font-size:12px">
set interfaces ae0 aggregated-ether-options bfd-liveness-detection minimum-interval 100
set interfaces ae0 aggregated-ether-options bfd-liveness-detection neighbor 2001:db8:1000::1
set interfaces ae0 aggregated-ether-options bfd-liveness-detection local-address 2001:db8:1000::2

set interfaces lo0 unit 0 family inet6 address 2001:db8:1000::2/128
</pre>


### Verifying our work

To verify the BFD sessions, we can use the following command:
<pre style="font-size:12px">
<b>show bfd session</b>
</pre>

To verify the LAG, we can use the following commands:<br>
<pre style="font-size:12px">
<b>show lacp interfaces ae0</b>
<b>show lacp statistics interfaces ae0</b>
<b>show interfaces ae0 extensive</b>
</pre>

Example output verifying the BFD sessions:

<pre style="font-size:12px">
salt@vMX-1> <b>show bfd session</b>
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
2001:db8:1000::2         Up        ge-0/0/7       0.300     0.100        3   
2001:db8:1000::2         Up        ge-0/0/6       0.300     0.100        3   
2001:db8:1000::2         Up        ge-0/0/5       0.300     0.100        3   
2001:db8:1000::2         Up        ge-0/0/4       0.300     0.100        3   

4 sessions, 4 clients
Cumulative transmit rate 40.0 pps, cumulative receive rate 40.0 pps
</pre>      

Example output verifying the AE interface:

<pre style="font-size:12px">
salt@vMX-1> <b>show lacp interfaces ae0</b>
Aggregated interface: ae0
< output omitted >
    LACP protocol:        Receive State  Transmit State          Mux State 
      ge-0/0/4                  Current   Fast periodic Collecting distributing
      ge-0/0/5                  Current   Fast periodic Collecting distributing
      ge-0/0/6                  Current   Fast periodic Collecting distributing
      ge-0/0/7                  Current   Fast periodic Collecting distributing

salt@vMX-1> <b>show lacp statistics interfaces ae0</b>
Aggregated interface: ae0
    LACP Statistics:       LACP Rx     LACP Tx   Unknown Rx   Illegal Rx 
      ge-0/0/4               23439       23456            0            0
      ge-0/0/5               23440       23457            0            0
      ge-0/0/6               23440       23457            0            0
      ge-0/0/7               23439       23457            0            0

salt@vMX-1> <b>show interfaces ae0 extensive</b>
Physical interface: ae0, Enabled, Physical link is Up
  Interface index: 158, SNMP ifIndex: 541, Generation: 174
  Description: vMX-2
  Link-level type: Ethernet, MTU: 1514, <b>Speed: 4Gbps</b>, BPDU Error: None, MAC-REWRITE Error: None, Loopback: Disabled, Source filtering: Disabled, Flow control: Disabled
  Pad to minimum frame size: Disabled
  Minimum links needed: 2, Minimum bandwidth needed: 1bps
< output omitted >
    <b>Aggregate member links: 4</b>

    <b>BFD View</b> :           Link        Usable
                   ge-0/0/4.0         Yes
                   ge-0/0/5.0         Yes
                   ge-0/0/6.0         Yes
                   ge-0/0/7.0         Yes
    <b>LACP info</b>:        Role     System             System       Port     Port    Port 
                             priority         identifier   priority   number     key 
      ge-0/0/4.0     Actor        127  2c:6b:f5:c3:d4:c0        127        1       1
      ge-0/0/4.0   Partner        127  2c:6b:f5:07:da:c0        127        2       1
      ge-0/0/5.0     Actor        127  2c:6b:f5:c3:d4:c0        127        2       1
      ge-0/0/5.0   Partner        127  2c:6b:f5:07:da:c0        127        1       1
      ge-0/0/6.0     Actor        127  2c:6b:f5:c3:d4:c0        127        3       1
      ge-0/0/6.0   Partner        127  2c:6b:f5:07:da:c0        127        3       1
      ge-0/0/7.0     Actor        127  2c:6b:f5:c3:d4:c0        127        4       1
      ge-0/0/7.0   Partner        127  2c:6b:f5:07:da:c0        127        4       1
    LACP Statistics:       LACP Rx     LACP Tx   Unknown Rx   Illegal Rx 
      ge-0/0/4.0                 0           0            0            0
      ge-0/0/5.0                 0           0            0            0
      ge-0/0/6.0                 0           0            0            0
      ge-0/0/7.0                 0           0            0            0
    Marker Statistics:   Marker Rx     Resp Tx   Unknown Rx   Illegal Rx
      ge-0/0/4.0                 0           0            0            0
      ge-0/0/5.0                 0           0            0            0
      ge-0/0/6.0                 0           0            0            0
      ge-0/0/7.0                 0           0            0            0 
< output omitted >
</pre>      




This is supported on QFX as well, or at least on QFX10k. The configuration is the same except for the way in which you add a child link to the LAG.

<b>MX</b>:

<pre style="font-size:12px">
set interfaces et-0/0/0 <font color='red'>gigether-options</font> 802.3ad ae0
</pre>

<b>QFX<b>:

<pre style="font-size:12px">
set interfaces et-0/0/0 <font color='red'>ether-options</font> 802.3ad ae0
</pre>