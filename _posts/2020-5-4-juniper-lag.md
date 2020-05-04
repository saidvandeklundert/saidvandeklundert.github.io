---
layout: post
title: Juniper BFD protected LAGs
tags: [ juniper ]
image: /img/juniper_logo.jpg
---


A LAG combines multiple physical links between two adjacent nodes together to establish a single (virtual) link. This offers increased bandwidth, link efficiency and physical redundancy.

In order to make protocols that run accross the LAG rapidly detect failures, you can configure them with BFD. BFD supports OSPF, IS-IS, BGP, LDP, RSVP, static routes and more. 

In some networks, BFD is configured for multiple protocols at the same time. But did you know that you can also protect a LAG using BFD? And that this protection also helps the higher-layer protocols to respond faster?

### Micro-BFD on LAG interfaces


When BFD is used as a liveness detection protocol for a LAG, micro-BFD sessions will monitor the forwarding path of the links between the two systems. The BFD sessions that are protecting the member links are independent BFD sessions. There is one BFD session per link that is part of the LAG. When BFD detects a failure in the path of a link, the child of the LAG is brought down. This way, BFD can detect failures in the forwarding path of a child link and ensure that it is brought down swiftly.

The BFD protected LAG will be able to respond to failures as fast as the BFD timers you configure. This in turn ensures that higher layer-protocols (such as OSPF, LDP or BGP) will be able to respond quickly to the loss of connectivity. This is because protocols running across interfaces will react nearly instantaneously to an interface down event. Consider an OSPF neighbor relationship with a dead timer of 40 seconds. When the underlying interface that is used to sustain the OSPF session is brought down by BFD, the system does not have to wait for the dead timer to reach 0. As soon as the LAG is brought down, the OSPF session is removed and alternate routes (if any) are considered. 

The same thing goes for other protocols and this makes it that the strategy can work out well for all sorts of networks, be it an MPLS core or a clos fabric.


### Configuring a LAG with BFD


In this example, we will configure a LAG between two MX routers:

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
<b>show bfd session extensive</b>
</pre>

To verify the LAG, we can use the following commands:<br>
<pre style="font-size:12px">
<b>show lacp interfaces ae0</b>
<b>show lacp statistics interfaces ae0</b>
<b>show interfaces ae0 extensive</b>
</pre>

Example output verifying the BFD sessions:

<pre style="font-size:12px">
salt@vMX-1> <b>show bfd session extensive</b>
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.0.2                 Up        <b>ge-0/0/4</b>       0.300     0.100        3   
 Client LACPD, TX interval 0.100, RX interval 0.100
 Session up time 06:34:00
 Local diagnostic None, remote diagnostic None
 Remote state Up, version 1
 Session type: Micro BFD
 Min async interval 0.100, min slow interval 1.000
 Adaptive async TX interval 0.100, RX interval 0.100
 Local min TX interval 0.100, minimum RX interval 0.100, multiplier 3
 Remote min TX interval 0.100, min RX interval 0.100, multiplier 3
 Local discriminator 21, remote discriminator 21
 Echo mode disabled/inactive
 Remote is control-plane independent
  Session ID: 0x0

                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.0.2                 Up        <b>ge-0/0/7</b>       0.300     0.100        3   
 Client LACPD, TX interval 0.100, RX interval 0.100
 Session up time 06:33:59
 Local diagnostic None, remote diagnostic None
 Remote state Up, version 1
 Session type: Micro BFD
 Min async interval 0.100, min slow interval 1.000
 Adaptive async TX interval 0.100, RX interval 0.100
 Local min TX interval 0.100, minimum RX interval 0.100, multiplier 3
 Remote min TX interval 0.100, min RX interval 0.100, multiplier 3
 Local discriminator 20, remote discriminator 20
 Echo mode disabled/inactive
 Remote is control-plane independent
  Session ID: 0x0

                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.0.2                 Up        <b>ge-0/0/6</b>       0.300     0.100        3   
 Client LACPD, TX interval 0.100, RX interval 0.100
 Session up time 06:34:00
 Local diagnostic None, remote diagnostic None
 Remote state Up, version 1
 Session type: Micro BFD
 Min async interval 0.100, min slow interval 1.000
 Adaptive async TX interval 0.100, RX interval 0.100
 Local min TX interval 0.100, minimum RX interval 0.100, multiplier 3
 Remote min TX interval 0.100, min RX interval 0.100, multiplier 3
 Local discriminator 19, remote discriminator 19
 Echo mode disabled/inactive            
 Remote is control-plane independent
  Session ID: 0x0

                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.0.2                 Up        <b>ge-0/0/5</b>       0.300     0.100        3   
 Client LACPD, TX interval 0.100, RX interval 0.100
 Session up time 06:34:00
 Local diagnostic None, remote diagnostic None
 Remote state Up, version 1
 Session type: Micro BFD
 Min async interval 0.100, min slow interval 1.000
 Adaptive async TX interval 0.100, RX interval 0.100
 Local min TX interval 0.100, minimum RX interval 0.100, multiplier 3
 Remote min TX interval 0.100, min RX interval 0.100, multiplier 3
 Local discriminator 18, remote discriminator 18
 Echo mode disabled/inactive
 Remote is control-plane independent
  Session ID: 0x0

4 sessions, 4 clients
Cumulative transmit rate 40.0 pps, cumulative receive rate 40.0 pps
</pre>      

Example output verifying the AE interface:

<pre style="font-size:12px">
salt@vMX-1> <b>show lacp interfaces ae0</b>
Aggregated interface: ae0
    LACP state:       Role   Exp   Def  Dist  Col  Syn  Aggr  Timeout  Activity
      ge-0/0/4       Actor    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/4     Partner    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/5       Actor    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/5     Partner    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/6       Actor    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/6     Partner    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/7       Actor    No    No   Yes  Yes  Yes   Yes     Fast    Active
      ge-0/0/7     Partner    No    No   Yes  Yes  Yes   Yes     Fast    Active
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
  Link-level type: Ethernet, MTU: 1514, Speed: 4Gbps, BPDU Error: None, MAC-REWRITE Error: None, Loopback: Disabled, Source filtering: Disabled, Flow control: Disabled
  Pad to minimum frame size: Disabled
  Minimum links needed: 2, Minimum bandwidth needed: 1bps
  Device flags   : Present Running
  Interface flags: SNMP-Traps Internal: 0x4000
  Current address: 2c:6b:f5:c3:d4:c0, Hardware address: 2c:6b:f5:c3:d4:c0
  Last flapped   : 2020-05-03 12:45:21 UTC (06:31:51 ago)
  Statistics last cleared: Never
  Traffic statistics:
   Input  bytes  :            191743826                22832 bps
   Output bytes  :             45822084                 4144 bps
   Input  packets:              3271894                   47 pps
   Output packets:               376862                    3 pps
   IPv6 transit statistics:
   Input  bytes  :                82112
   Output bytes  :                    0
   Input  packets:                 1252
   Output packets:                    0
  Label-switched interface (LSI) traffic statistics:
   Input  bytes  :                    0                    0 bps
   Input  packets:                    0                    0 pps
  Dropped traffic statistics due to STP State:
   Input  bytes  :                    0
   Output bytes  :                    0
   Input  packets:                    0
   Output packets:                    0
  MAC statistics:                      Receive         Transmit
    Broadcast packets                        0                0
    Multicast packets                        0                0
  Input errors:
    Errors: 0, Drops: 0, Framing errors: 0, Runts: 0, Giants: 0, Policed discards: 0, Resource errors: 0
  Output errors:
    Carrier transitions: 4, Errors: 0, Drops: 0, MTU errors: 0, Resource errors: 0
  Ingress queues: 8 supported, 4 in use
  Queue counters:       Queued packets  Transmitted packets      Dropped packets
    0                                0                    0                    0
    1                                0                    0                    0
    2                                0                    0                    0
    3                                0                    0                    0
  Egress queues: 8 supported, 4 in use
  Queue counters:       Queued packets  Transmitted packets      Dropped packets
    0                                0                    0                    0
    1                                0                    0                    0
    2                                0                    0                    0
    3                                0                    0                    0
  Queue number:         Mapped forwarding classes
    0                   best-effort
    1                   expedited-forwarding
    2                   assured-forwarding
    3                   network-control

  Logical interface ae0.0 (Index 329) (SNMP ifIndex 548) (Generation 169)
    Flags: Up SNMP-Traps 0x4004000 Encapsulation: ENET2
    Statistics        Packets        pps         Bytes          bps
    Bundle:
        Input :       1044439         47      54547394        18896
        Output:          9008          0        765736            0
    Adaptive Statistics:
        Adaptive Adjusts:          0
        Adaptive Scans  :          0
        Adaptive Updates:          0
    Link:
      ge-0/0/4.0
        Input :        258264         12      13429736         4704
        Output:          9008          0        765736            0
      ge-0/0/5.0
        Input :        269916         12      14272362         4984
        Output:             0          0             0            0
      ge-0/0/6.0
        Input :        258144         12      13423402         4704
        Output:             0          0             0            0
      ge-0/0/7.0
        Input :        258115         11      13421894         4504
        Output:             0          0             0            0


    Aggregate member links: 4

    BFD View :           Link        Usable
                   ge-0/0/4.0         Yes
                   ge-0/0/5.0         Yes
                   ge-0/0/6.0         Yes
                   ge-0/0/7.0         Yes
    LACP info:        Role     System             System       Port     Port    Port 
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
    List-Type       Status
    Primary         Active
                Interfaces:
                ge-0/0/4        Up   
                ge-0/0/5        Up   
                ge-0/0/6        Up   
                ge-0/0/7        Up   
    List-Type       Status
    Backup          Down 
    Standby         Down 
    Protocol inet, MTU: 1500
    Max nh cache: 75000, New hold nh limit: 75000, Curr nh cnt: 1, Curr new hold cnt: 0, NH drop cnt: 0
    Generation: 221, Route table: 0
      Flags: Sendbcast-pkt-to-re
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 10.100.0.0/31, Local: 10.100.0.0, Broadcast: Unspecified, Generation: 214
    Protocol inet6, MTU: 1500
    Max nh cache: 75000, New hold nh limit: 75000, Curr nh cnt: 1, Curr new hold cnt: 0, NH drop cnt: 0
    Generation: 222, Route table: 0
      Flags: Is-Primary
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 2001:db8:1000::/127, Local: 2001:db8:1000::
    Generation: 216
      Addresses, Flags: Is-Preferred
        Destination: fe80::/64, Local: fe80::2e6b:f5ff:fec3:d4c0
    Protocol multiservice, MTU: Unlimited, Generation: 218
    Generation: 224, Route table: 0
      Flags: Is-Primary
      Policer: Input: __default_arp_policer__
</pre>      




This is supported on QFX as well, or at least on QFX10k. The configuration is the same save for the way in which you add the child link.

<b>MX</b>:

<pre style="font-size:12px">
set interfaces et-0/0/0 <font color='red'>gigether-options</font> 802.3ad ae0
</pre>

<b>QFX<b>:

<pre style="font-size:12px">
set interfaces et-0/0/0 <font color='red'>ether-options</font> 802.3ad ae0
</pre>