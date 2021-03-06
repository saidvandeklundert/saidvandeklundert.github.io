---
layout: post
title: MPLS L3VPN between Juniper MX and Cisco IOS XR
tags: [cisco, juniper, mpls, vpn]
image: /img/cisco_juniper_logo.png
---

Lately, I have been playing around with a lab that involves Cisco IOS XR and Juniper devices. The main intention I have with it is to be able to quickly test something or to check how I could automate something.

The topology I am using is the following:

![juniper and ios xr topology](/img/topology_juniper_ios_xr.png "topology")

Nodes in this small SP lab have the following functions:
- rr: `vmx14` and `vmx15`
- p: `vmx1` - `vmx4`
- pe: `ios_xr_1`, `ios_xr_2`, `vmx5` and `vmx6`

In this post, I will walk you through the following configuration and verification:
- Interfaces
- OSPF IGP configuration with p2p links, authentication, BFD and load-balancing
- MPLS LDP with authentication and LDP synchronization
- BGP sessions that supports VPN routes and MD5 authentication
- MPLS L3VPN with static routes


<br>

Interface configuration
=======================

The interface, OSPF and LDP configuration is going to be the same on every device. For that reason, the example configuration for these sections will include the configuration and verification steps on `ios_xr_1` and `vmx1`. 

The `vmx1` and `ios_xr_1` device are connected together using the following interfaces:

![ios_xr_1 to vmx1](/img/juniper_ios_xr.png "ios_xr_1 to vmx1")

First we configure the interfaces on `ios_xr_1`:

```
interface Loopback0
 ipv4 address 10.0.1.1 255.255.255.255

interface GigabitEthernet0/0/0/1
 no shutdown 
 
interface GigabitEthernet0/0/0/1.10
 description vmx1
 ipv4 address 10.0.2.1 255.255.255.254
 encapsulation dot1q 1
```

Then we move on to `vmx1`:

```
set interfaces ge-0/0/3 flexible-vlan-tagging
set interfaces ge-0/0/3 encapsulation flexible-ethernet-services
set interfaces ge-0/0/3 unit 10 description ios_xr_1
set interfaces ge-0/0/3 unit 10 vlan-id 10
set interfaces ge-0/0/3 unit 10 family inet address 10.0.2.0/31

set interfaces lo0 unit 1 family inet address 10.0.0.1/32
```

Apart from the obvious difference in syntax, we notice two things. The Cisco interfaces need to be enabled using `no shutdown` before they can be used. On the Juniper device, we need to ready the interface to allow different encapsulation on the main interface as well as on the sub-interface. Because `flexible-vlan-tagging` and `encapsulation flexible-ethernet-services` will enable most of the scenario's, I generally default to using that.

To verify the interface status on `ios_xr_1`, we issue to the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show interface GigabitEthernet0/0/0/1.10</b>
GigabitEthernet0/0/0/1.10 is up, line protocol is up 
  Interface state transitions: 1
  Hardware is VLAN sub-interface(s), address is 5246.8457.a004
  Description: vmx1
  Internet address is 10.0.2.1/31
  MTU 1518 bytes, BW 1000000 Kbit (Max: 1000000 Kbit)
     reliability 255/255, txload 0/255, rxload 0/255
  Encapsulation 802.1Q Virtual LAN, VLAN Id 10,  loopback not set,
  Last link flapped 01:32:09
  ARP type ARPA, ARP timeout 04:00:00
  Last input 00:00:00, output 00:00:00
  Last clearing of "show interface" counters never
  5 minute input rate 6000 bits/sec, 11 packets/sec
  5 minute output rate 6000 bits/sec, 11 packets/sec
     58949 packets input, 4244122 bytes, 130 total input drops
     0 drops for unrecognized upper-level protocol
     Received 9 broadcast packets, 1939 multicast packets
     62790 packets output, 4514492 bytes, 0 total output drops
     Output 2 broadcast packets, 1819 multicast packets
</pre>


To verify the interface on `vmx1`:

<pre>
salt@vmx1> <b>show interfaces ge-0/0/3.10</b>  
  Logical interface ge-0/0/3.10 (Index 414) (SNMP ifIndex 584)
    Description: ios_xr_1
    Flags: Up SNMP-Traps 0x4000 VLAN-Tag [ 0x8100.10 ]  Encapsulation: ENET2
    Input packets : 1578085
    Output packets: 1575117
    Protocol inet, MTU: 1500
    Max nh cache: 75000, New hold nh limit: 75000, Curr nh cnt: 1, Curr new hold cnt: 0, NH drop cnt: 0
      Flags: Sendbcast-pkt-to-re
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 10.0.2.0/31, Local: 10.0.2.0
    Protocol multiservice, MTU: Unlimited
</pre>  

To wrap up interfaces configuration, we verify that we are able to ping across the interface:

<pre>
salt@vmx1> <b>ping 10.0.2.1 rapid</b> 
PING 10.0.2.1 (10.0.2.1): 56 data bytes
!!!!!
--- 10.0.2.1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.581/4.794/5.613/0.668 ms
</pre>



Configuring and verifying the IGP
=================================

What will be included in the IGP configuration is the following:
- OSPF area 0 interface configuration
- Take into account an OSPF reference bandwidth of 100G
- Authentication
- BFD
- Load-balancing

The following is the `ios_xr_1` configuration:

```
router ospf 1
 router-id 10.0.1.1
 auto-cost reference-bandwidth 100000
 area 0
  interface Loopback0
   passive enable
  !
  interface GigabitEthernet0/0/0/1.10
   bfd minimum-interval 100
   bfd fast-detect
   bfd multiplier 5
   authentication message-digest
   message-digest-key 1 md5 encrypted 04480A0A1B701E1D
   network point-to-point
  !
```


The equivalent configuration for `vmx1` is as follows:

```
set protocols ospf reference-bandwidth 100g
set protocols ospf area 0.0.0.0 interface lo0.1 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 authentication md5 1 key "$9$HmQn/9p1RStuWL7NbwP5T"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 bfd-liveness-detection minimum-interval 100
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 bfd-liveness-detection multiplier 5
```

Since load balancing is not something that is automatically enabled on Juniper devices, we need to add the following to `vmx1`:

```
set policy-options policy-statement lbpp term 1 then load-balance per-packet
set routing-options forwarding-table export lbpp
```

Do not let the `load-balance per-packet` name fool you. This action will have the Juniper load balance traffic per flow.

After the configuration is applied on both devices, we can start our verification. 

First, we check the OSPF neighbor status on both sides.

On the Cisco, we check the following:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show ospf neighbor 10.0.0.1</b>

* Indicates MADJ interface
# Indicates Neighbor awaiting BFD session up

Neighbors for OSPF 1

 Neighbor 10.0.0.1, interface address 10.0.2.0
    In the area 0 via interface GigabitEthernet0/0/0/1.10 , BFD enabled, Mode: Default
    Neighbor priority is 128, State is <b>FULL</b>, 7 state changes
    DR is 0.0.0.0 BDR is 0.0.0.0
    Options is 0x52  
    LLS Options is 0x1 (LR)
    Dead timer due in 00:00:37
    Neighbor is up for 23:05:08
    Number of DBD retrans during last exchange 0
    Index 2/2, retransmission queue length 0, number of retransmission 18
    First 0(0)/0(0) Next 0(0)/0(0)
    Last retransmission scan length is 1, maximum is 2
    Last retransmission scan time is 0 msec, maximum is 0 msec
    LS Ack list: NSR-sync pending 0, high water mark 0
    <b>Neighbor BFD status: Session up</b>
</pre>

The neighbor state is `FULL` and the BFD session that provides us with fast failure detection is also up. Other details can be verified by issuing the `show ospf 1 interface` command, like so:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show ospf 1 interface GigabitEthernet0/0/0/1.10</b>           

GigabitEthernet0/0/0/1.10 is up, line protocol is up 
  Internet Address 10.0.2.1/31, Area 0
  Label stack Primary label 1 Backup label 3 SRTE label 10
  Process ID 1, Router ID 10.0.1.1, Network Type <b>POINT_TO_POINT</b>, <b>Cost: 100</b>
  Transmit Delay is 1 sec, State POINT_TO_POINT, MTU 1500, MaxPktSz 1500
  Forward reference No, Unnumbered no,  Bandwidth 1000000 
  <b>BFD enabled, BFD interval 100 msec, BFD multiplier 5, Mode: Default</b>
  Timer intervals configured, Hello 10, Dead 40, Wait 40, Retransmit 5
    Hello due in 00:00:07:661
  Index 2/2, flood queue length 0
  Next 0(0)/0(0)
  Last flood scan length is 1, maximum is 4
  Last flood scan time is 0 msec, maximum is 0 msec
  LS Ack List: current length 0, high water mark 22
  Neighbor Count is 1, Adjacent neighbor count is 1
    Adjacent with neighbor 10.0.0.1
  Suppress hello for 0 neighbor(s)
  <b>Message digest authentication enabled</b>
    Youngest key id is 1
  Multi-area interface Count is 0
</pre>

Here we see that the interface is of the type `POINT_TO_POINT` and that the link has a cost of 100. We can also see the BFD settings here. And lastly, we see what authentication is configured.

In case we really wanted to know about all the details on the BFD session, we should turn to the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show bfd session interface GigabitEthernet0/0/0/1.10 detail</b>
I/f: GigabitEthernet0/0/0/1.10, Location: 0/0/CPU0
Dest: 10.0.2.0
Src: 10.0.2.1
 State: UP for 0d:0h:27m:40s, number of times UP: 1
 Session type: PR/V4/SH
Received parameters:
 Version: 1, desired tx interval: 100 ms, required rx interval: 100 ms
 Required echo rx interval: 0 ms, multiplier: 5, diag: None
 My discr: 32, your discr: 2148532225, state UP, D/F/P/C/A: 0/0/0/1/0
Transmitted parameters:
 Version: 1, desired tx interval: 100 ms, required rx interval: 100 ms
 Required echo rx interval: 1 ms, multiplier: 5, diag: None
 My discr: 2148532225, your discr: 32, state UP, D/F/P/C/A: 0/0/0/1/0
Timer Values:
 Local negotiated async tx interval: 100 ms
 Remote negotiated async tx interval: 100 ms
 Desired echo tx interval: 100 ms, local negotiated echo tx interval: 0 ms
 Echo detection time: 0 ms(0 ms*5), async detection time: 500 ms(100 ms*5)
Local Stats:
 Intervals between async packets:
   Tx: Number of intervals=100, min=1 ms, max=112 ms, avg=47 ms
       Last packet transmitted 71 ms ago
   Rx: Number of intervals=100, min=81 ms, max=119 ms, avg=100 ms
       Last packet received 89 ms ago
 Intervals between echo packets:
   Tx: Number of intervals=0, min=0 s, max=0 s, avg=0 s
       Last packet transmitted 0 s ago
   Rx: Number of intervals=0, min=0 s, max=0 s, avg=0 s
       Last packet received 0 s ago
 Latency of echo packets (time between tx and rx):
   Number of packets: 0, min=0 ms, max=0 ms, avg=0 ms
Session owner information:
                            Desired               Adjusted
  Client               Interval   Multiplier Interval   Multiplier
  -------------------- --------------------- ---------------------
  ospf-1               100 ms     5          100 ms     5     
</pre> 


Let's verify the same things on the Juniper side and start off checking the OSPF neighbor state:

<pre>
salt@vmx1> <b>show ospf neighbor 10.0.1.1 extensive</b> 
Address          Interface              State     ID               Pri  Dead
10.0.2.1         ge-0/0/3.10            <b>Full</b>      <b>10.0.1.1</b>           1    37
  Area 0.0.0.0, opt 0x52, DR 0.0.0.0, BDR 0.0.0.0
  Up 23:08:38, adjacent 23:08:38
  Topology default (ID 0) -> Bidirectional
</pre>

The neighbor state is `Full`. To verify some of the other features we configured, we check the following:

<pre>
salt@vmx1> <b>show ospf interface ge-0/0/3.10 extensive</b>
Interface           State   Area            DR ID           BDR ID          Nbrs
ge-0/0/3.10         PtToPt  0.0.0.0         0.0.0.0         0.0.0.0            1
  Type: <b>P2P</b>, Address: 10.0.2.0, Mask: 255.255.255.254, MTU: 1500, Cost: 1
  Adj count: 1
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  <b>Auth type: MD5</b>, Active key ID: 1, Start time: 1970 Jan  1 00:00:00 UTC
  Protection type: None
  Topology default (ID 0) -> <b>Cost: 100</b>
</pre>

The previous output reveals the interface is acting as a `P2P` OSPF interface, we have `MD5` authentication enabled and the cost for the link is `100`.

BFD can be verified using the following:

<pre>
salt@vmx1> <b>show bfd session extensive</b>                
                                                  Detect   Transmit
Address                  State     Interface      Time     Interval  Multiplier
10.0.2.1                 Up        ge-0/0/3.10    0.500     0.100        5   
 Client OSPF realm ospf-v2 Area 0.0.0.0, TX interval 0.100, RX interval 0.100
 Session up time 00:21:55
 Local diagnostic None, remote diagnostic None
 Remote state Up, version 1
 Logical system 1, routing table index 9
 Session type: Single hop BFD
 Min async interval 0.100, min slow interval 1.000
 Adaptive async TX interval 0.100, RX interval 0.100
 Local min TX interval 0.100, minimum RX interval 0.100, multiplier 5
 Remote min TX interval 0.100, min RX interval 0.100, multiplier 5
 Local discriminator 17, remote discriminator 2148532225
 Echo mode disabled/inactive
 Remote is control-plane independent
  Session ID: 0x16f
</pre> 

Both routers should have an OSPF route towards their neighboring loopback IP now. We can verify things on `ios_xr_1` like so:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show route 10.0.0.1</b>

Routing entry for 10.0.0.1/32
  Known via "ospf 1", distance 110, metric 100, type intra area
  Installed Aug 26 08:23:57.320 for 00:04:51
  Routing Descriptor Blocks
    10.0.2.0, from 10.0.0.1, via GigabitEthernet0/0/0/1.10
      Route metric is 100
  No advertising protos.  
</pre>

On `vmx1`, we do the following:

<pre>
salt@vmx1> <b>show route 10.0.1.1</b>   

inet.0: 30 destinations, 30 routes (30 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.1.1/32        *[OSPF/10] 00:04:24, metric 101
                    >  to 10.0.2.1 via ge-0/0/3.10
</pre>                   

Notice the difference in cost. Both Juniper and Cisco assigned a cost of 100 to the link, but the cost assigned to the loopback interface differs. 

We can see the following on `ios_xr_1`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show ospf interface loopback 0</b>

Loopback0 is up, line protocol is up 
  Internet Address 10.0.1.1/32, Area 0
  Label stack Primary label 0 Backup label 0 SRTE label 0
  Process ID 1, Router ID 10.0.1.1, Network Type LOOPBACK, <b>Cost: 1</b>
  Loopback interface is treated as a stub Host
</pre>

On `vmx1`, we see the following:

<pre>
salt@vmx1> <b>show ospf interface lo0.1 detail</b> 
Interface           State   Area            DR ID           BDR ID          Nbrs
lo0.1               DRother 0.0.0.0         0.0.0.0         0.0.0.0            0
  Type: LAN, Address: 10.0.0.1, Mask: 255.255.255.255, MTU: 65535, <b>Cost: 0</b>
  Adj count: 0, Passive
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  Auth type: None
  Protection type: None
  Topology default (ID 0) -> Passive, Cost: 0
</pre>


After enabling the same OSPF configuration on every device in our topology, we can check the route table to see if we have an OSPF route towards the loopback interface of every other device.

On the `ios_xr_1`, we issue the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show route ipv4 ospf | include 32</b> 
O    10.0.0.1/32 [110/100] via 10.0.2.0, 00:15:27, GigabitEthernet0/0/0/1.10
O    10.0.0.2/32 [110/100] via 10.0.2.4, 00:02:24, GigabitEthernet0/0/0/2.12
O    10.0.0.3/32 [110/200] via 10.0.2.4, 00:02:24, GigabitEthernet0/0/0/2.12
O    10.0.0.4/32 [110/200] via 10.0.2.0, 00:15:27, GigabitEthernet0/0/0/1.10
O    10.0.0.5/32 [110/300] via 10.0.2.4, 00:02:24, GigabitEthernet0/0/0/2.12
O    10.0.0.6/32 [110/300] via 10.0.2.4, 00:02:24, GigabitEthernet0/0/0/2.12
O    10.0.0.14/32 [110/300] via 10.0.2.0, 00:15:27, GigabitEthernet0/0/0/1.10
O    10.0.0.15/32 [110/200] via 10.0.2.0, 00:15:27, GigabitEthernet0/0/0/1.10
O    10.0.1.2/32 [110/201] via 10.0.2.4, 00:02:24, GigabitEthernet0/0/0/2.12
</pre>

On `vmx1`, we issue the following command:

<pre>
salt@vmx1> <b>show route protocol ospf | match 32</b>
10.0.0.2/32        *[OSPF/10] 00:00:05, metric 100
10.0.0.3/32        *[OSPF/10] 00:00:05, metric 200
10.0.0.4/32        *[OSPF/10] 00:00:05, metric 100
10.0.0.5/32        *[OSPF/10] 00:00:05, metric 200
10.0.0.6/32        *[OSPF/10] 00:00:05, metric 200
10.0.0.14/32       *[OSPF/10] 00:00:05, metric 200
10.0.0.15/32       *[OSPF/10] 00:00:05, metric 100
10.0.1.1/32        *[OSPF/10] 00:00:05, metric 101
10.0.1.2/32        *[OSPF/10] 00:00:05, metric 101
</pre>

The last thing we can do is verify load-balancing. Since there is only 1 link between `ios_xr_1` and `vmx1`, we check the routes towards a node in the network to which the devices have multiple equal cost paths. On the Cisco, we check the route towards `ios_xr_2`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show route 10.0.1.2</b>

Routing entry for 10.0.1.2/32
  Known via "ospf 1", distance 110, metric 201, type intra area
  Installed Aug 20 08:52:01.911 for 00:00:29
  Routing Descriptor Blocks
    10.0.2.0, from 10.0.1.2, via GigabitEthernet0/0/0/1.10
      Route metric is 201
    10.0.2.4, from 10.0.1.2, via GigabitEthernet0/0/0/2.12
      Route metric is 201
  No advertising protos.
</pre>

On `vmx1`, we check the route towards `vmx3`:

<pre>
salt@vmx1> <b>show route 10.0.0.3</b>                          

inet.0: 30 destinations, 30 routes (30 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.0.3/32        *[OSPF/10] 00:00:12, metric 200
                       to 192.168.4.1 via ge-0/0/1.4
                    >  to 192.168.1.1 via ge-0/0/1.1
</pre>

Here we see two equal cost routes are available. To see if they are actually being used, we need to issue the following command:

<pre>
salt@vmx1> <b>show route forwarding-table destination 10.0.0.3</b>   
Logical system: r1
Routing table: default.inet
Internet:
Enabled protocols: Bridging, Dual VLAN, 
Destination        Type RtRef Next hop           Type Index    NhRef Netif
10.0.0.3/32        user     0                    <b>ulst</b>  1048587     4
                              192.168.4.1        ucst      843    12 ge-0/0/1.4
                              192.168.1.1        ucst      842     8 ge-0/0/1.1
</pre>

The `ulst` is indicative of a `List of unicast next hops` that will be used to forward traffic.


<br>


Configuring and verifying LDP
=============================

In order to get any MPLS L3VPN going, we require an MPLS network. We'll take the easy route and settle for LDP. The LDP configuration will include the following:
- authentication
- advertisement of the loopback IP only

Additionally, we want to ensure that the IGP metrics are reflected in the LDP routes on the Juniper device.

The LDP configuration is the same on every router, so again, we focus on the configuration between `ios_xr_1` and `vmx1`. First, we check the MPLS configuration on `ios_xr_1`:

```
mpls ldp
 router-id 10.0.1.1
 neighbor
  password encrypted 15010A00107B7977
 !
 address-family ipv4
  label
   local
    allocate for host-routes
    !
   !
  !
 !
 interface GigabitEthernet0/0/0/1.10
 !
!
router ospf 1
 area 0
  interface GigabitEthernet0/0/0/1.10
   mpls ldp sync
  !
 !
!
```

Now the configuration on `vmx1`:

```
set interfaces ge-0/0/3 unit 10 family mpls

set protocols mpls interface ge-0/0/3.10

set protocols ldp track-igp-metric
set protocols ldp interface ge-0/0/3.10
set protocols ldp session-group 0.0.0.0/0 authentication-key "$9$.539AtOEcl0BX7dVY2TzF"

set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 ldp-synchronization
```

Notice how specifying the interfaces under LDP is enough for IOS XR. For the Juniper, configuring the interfaces under LDP is not enough. We also need to enable the interfaces for MPLS processing by specifying the MPLS address family under the interface configuration and under `protocols mpls`.

Another thing worth knowing is that the defaults for Juniper and Cisco are slightly different. 

For Juniper, we configured the `track-igp-metric` option. This will copy the IGP metric into the `inet.3` table. Normally, Juniper will default all LDP routes to 1.

For Cisco, we configured the `mpls ldp address-family ipv4 label local allocate for host-routes` option. This causes for the IOS XR router to advertise a label for the lo0 interface only. Unlike on a Juniper, the default is to advertise a label for every interface.

The other configuration items are pretty similar. Though different in syntax, on both Juniper and Cisco we configure the device to:
- enable LDP synchronization by specifying that under the OSPF stanza
- authenticate all LDP sessions through a single configuration command

Both Cisco as well as Juniper offer ways to inject into LDP whatever routes you want, both offer BFD protection and many more configuration options, but I do not want to go off to far into the weeds here. Instead, we will verify what we have configured so far.

First, we check the LDP adjacency state on Cisco:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show mpls ldp discovery</b> 

Local LDP Identifier: 10.0.1.1:0
Discovery Sources:
  Interfaces:
    GigabitEthernet0/0/0/1.10 : xmit/recv
      VRF: 'default' (0x60000000)
      LDP Id: 10.0.0.1:0, Transport address: 10.0.0.1
          Hold time: 15 sec (local:15 sec, peer:15 sec)
          Established: Aug 20 11:56:03.380 (00:00:53 ago)

</pre>

Next on Juniper:

<pre>
salt@vmx1> <b>show ldp neighbor</b>           
Address                             Interface       Label space ID     Hold time
10.0.2.1                            ge-0/0/3.10     10.0.1.1:0           10
</pre>

After forming this adjacency, an LDP session is established. To verify the LDP session, we issue the following command on the Cisco device:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show mpls ldp neighbor</b>

Peer LDP Identifier: 10.0.0.1:0
  TCP connection: 10.0.0.1:646 - 10.0.1.1:33317; MD5 on
  Graceful Restart: No
  Session Holdtime: 30 sec
  State: Oper; Msgs sent/rcvd: 25/23; Downstream-Unsolicited
  Up time: 00:01:52
  LDP Discovery Sources:
    IPv4: (1)
      GigabitEthernet0/0/0/1.10
    IPv6: (0)
  Addresses bound to this peer:
    IPv4: (5)
      10.0.2.0       10.0.2.2       192.168.1.0    192.168.4.0    
      192.168.15.0   
    IPv6: (0)

</pre>

And on the Juniper device:

<pre>
salt@vmx1> <b>show ldp session detail</b> 
Address: 10.0.1.1, State: Operational, Connection: Open, Hold time: 29
  Session ID: 10.0.0.1:0--10.0.1.1:0
  Next keepalive in 7 seconds
  Passive, Maximum PDU: 4096, Hold time: 30, Neighbor count: 1
  Neighbor types: discovered
  Keepalive interval: 10, Connect retry interval: 1
  Local address: 10.0.0.1, Remote address: 10.0.1.1
  Up for 00:02:32
  Capabilities advertised: none
  Capabilities received: p2mp
  Protection: disabled
  Session flags: none
  Authentication type: MD5 (0.0.0.0/0)
  Local - Restart: disabled, Helper mode: enabled
  Remote - Restart: disabled, Helper mode: disabled
  Local maximum neighbor reconnect time: 120000 msec
  Local maximum neighbor recovery time: 240000 msec
  Local Label Advertisement mode: Downstream unsolicited
  Remote Label Advertisement mode: Downstream unsolicited
  Negotiated Label Advertisement mode: Downstream unsolicited
  MTU discovery: disabled
  Nonstop routing state: Not in sync
  Next-hop addresses received:
    10.0.1.1                            
    10.0.2.1
    10.0.2.5
</pre>

Next we check the LDP synchronization. It was configured under OSPF, and on both Cisco as well as Juniper, that is the place where we verify it as well. First, we check the Cisco:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show ospf interface GigabitEthernet0/0/0/1.10 | include LDP</b>
  LDP Sync Enabled, Sync Status: Achieved
</pre>

Then we check the Juniper:

<pre>
salt@vmx1> <b>show ospf interface ge-0/0/3.10 extensive | match ldp</b> 
  LDP sync state: in sync, for: 00:13:39, reason: LDP session up
</pre>

If all is well, both devices have now signaled an LDP LSP between them. Let's verify this on the Cisco side:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show cef 10.0.0.1</b>
10.0.0.1/32, version 7, internal 0x1000001 0x0 (ptr 0xdf13d88) [1], 0x0 (0xe0d79e8), 0xa20 (0xe710228)
 remote adjacency to GigabitEthernet0/0/0/1.10
 Prefix Len 32, traffic index 0, precedence n/a, priority 3
   via 10.0.2.0/32, GigabitEthernet0/0/0/1.10, 8 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0xf141140 0xf141380]
    next hop 10.0.2.0/32
    remote adjacency
     local label 24002      labels imposed {ImplNull}

RP/0/RP0/CPU0:ios_xr_1#<b>show mpls ldp ipv4 forwarding 10.0.0.1/32</b>

Codes: 
  - = GR label recovering, (!) = LFA FRR pure backup path 
  {} = Label stack with multi-line output for a routing path
  G = GR, S = Stale, R = Remote LFA FRR backup

Prefix          Label   Label(s)       Outgoing     Next Hop            Flags
                In      Out            Interface                        G S R
--------------- ------- -------------- ------------ ------------------- -----
10.0.0.1/32     24002   ImpNull        Gi0/0/0/1.10 10.0.2.0                       
</pre>

And now on the Juniper side:

<pre>
salt@vmx1> <b>show route 10.0.1.2/32</b>

inet.0: 31 destinations, 40 routes (31 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.1.2/32        @[OSPF/10] 00:03:22, metric 101
                    >  to 10.0.2.3 via ge-0/0/3.11
                   #[LDP/9] 00:02:48, metric 1
                    >  to 10.0.2.3 via ge-0/0/3.11

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.2/32        *[LDP/9] 00:02:48, metric 1
                    >  to 10.0.2.3 via ge-0/0/3.11
</pre>


The LDP configuration is the same for every device in the topology. After enabling the configuration on every device, we can start verifying whether or not we have all the required routes.

We can check this on the Cisco using the following commands:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show mpls ldp ipv4 forwarding</b>

Codes: 
  - = GR label recovering, (!) = LFA FRR pure backup path 
  {} = Label stack with multi-line output for a routing path
  G = GR, S = Stale, R = Remote LFA FRR backup

Prefix          Label   Label(s)       Outgoing     Next Hop            Flags
                In      Out            Interface                        G S R
--------------- ------- -------------- ------------ ------------------- -----
10.0.0.1/32     24002   ImpNull        Gi0/0/0/1.10 10.0.2.0                 
10.0.0.2/32     24000   ImpNull        Gi0/0/0/2.12 10.0.2.4                 
10.0.0.3/32     24001   338            Gi0/0/0/2.12 10.0.2.4                 
10.0.0.4/32     24003   175            Gi0/0/0/1.10 10.0.2.0                 
10.0.0.5/32     24004   176            Gi0/0/0/1.10 10.0.2.0                 
                        340            Gi0/0/0/2.12 10.0.2.4                 
10.0.0.6/32     24005   177            Gi0/0/0/1.10 10.0.2.0                 
                        341            Gi0/0/0/2.12 10.0.2.4                 
10.0.0.14/32    24006   178            Gi0/0/0/1.10 10.0.2.0                 
10.0.0.15/32    24007   172            Gi0/0/0/1.10 10.0.2.0                 
10.0.1.2/32     24008   182            Gi0/0/0/1.10 10.0.2.0                 
                        348            Gi0/0/0/2.12 10.0.2.4                 
</pre>

Next, we check the the same thing on the Juniper:

<pre>
salt@vmx1> <b>show route protocol ldp table inet.3</b> 

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] 00:14:32, metric 1
                    >  to 192.168.1.1 via ge-0/0/1.1
10.0.0.3/32        *[LDP/9] 00:14:32, metric 1
                       to 192.168.4.1 via ge-0/0/1.4, Push 316
                    >  to 192.168.1.1 via ge-0/0/1.1, Push 338
10.0.0.4/32        *[LDP/9] 00:14:32, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4
10.0.0.5/32        *[LDP/9] 00:14:32, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 318
10.0.0.6/32        *[LDP/9] 00:14:32, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 319
10.0.0.14/32       *[LDP/9] 00:14:32, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 320
10.0.0.15/32       *[LDP/9] 00:14:32, metric 1
                    >  to 192.168.15.1 via ge-0/0/1.15
10.0.1.1/32        *[LDP/9] 00:05:04, metric 1
                    >  to 10.0.2.1 via ge-0/0/3.10
10.0.1.2/32        *[LDP/9] 00:04:02, metric 1
                    >  to 10.0.2.3 via ge-0/0/3.11
</pre>

In case we wanted to see the labels advertised/received acrros the LDP session, we can use the following command on `ios_xr_1`:
<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show mpls ldp bindings neighbor 10.0.0.1</b>
Mon Aug 26 20:27:15.289 UTC

10.0.0.1/32, rev 16
        Local binding: label: 24002
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          ImpNull 
10.0.0.2/32, rev 12
        Local binding: label: 24000
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          173     
10.0.0.3/32, rev 14
        Local binding: label: 24001
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          174     
10.0.0.4/32, rev 18
        Local binding: label: 24003
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          175     
10.0.0.5/32, rev 20
        Local binding: label: 24004
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          176     
10.0.0.6/32, rev 22
        Local binding: label: 24005
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          177     
10.0.0.14/32, rev 24
        Local binding: label: 24006
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          178     
10.0.0.15/32, rev 26
        Local binding: label: 24007
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          172     
10.0.1.1/32, rev 10
        Local binding: label: ImpNull
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          181     
10.0.1.2/32, rev 28
        Local binding: label: 24008
        Remote bindings: (2 peers)
            Peer                Label    
            -----------------   ---------
            10.0.0.1:0          182     
</pre>        

On `vmx1`, we can issue the following command:

<pre>
salt@vmx01:r1> <b>show ldp database session 10.0.1.1</b>
Input label database, 10.0.0.1:0--10.0.1.1:0
Labels received: 10
  Label     Prefix
  24002      10.0.0.1/32
  24000      10.0.0.2/32
  24001      10.0.0.3/32
  24003      10.0.0.4/32
  24004      10.0.0.5/32
  24005      10.0.0.6/32
  24006      10.0.0.14/32
  24007      10.0.0.15/32
      3      10.0.1.1/32
  24008      10.0.1.2/32

Output label database, 10.0.0.1:0--10.0.1.1:0
Labels advertised: 10
  Label     Prefix
      3      10.0.0.1/32
    173      10.0.0.2/32
    174      10.0.0.3/32
    175      10.0.0.4/32
    176      10.0.0.5/32
    177      10.0.0.6/32
    178      10.0.0.14/32
    172      10.0.0.15/32
    181      10.0.1.1/32
    182      10.0.1.2/32
</pre>

Additional verification of label information will be done when the VPN configuration is completed.


<br>

Configuring and verifying BGP
=============================

In this section, we will configure the following BGP sessions:

![juniper and ios xr BGP configuration](/img/rr_bgp_configuration.png "BGP configuration")

We will enable the BGP session to carry VPN routes and we will authenticate the sessions.

Let's examine the configuration on the route reflectors first.

On `vmx14`:

```
set protocols bgp group rr type internal
set protocols bgp group rr local-address 10.0.0.14
set protocols bgp group rr family inet-vpn unicast
set protocols bgp group rr authentication-key "$9$SKwe87-dsoJDwYP5zF/9KMW"
set protocols bgp group rr cluster 0.0.0.1
set protocols bgp group rr neighbor 10.0.1.1
set protocols bgp group rr neighbor 10.0.1.2
set protocols bgp group rr neighbor 10.0.0.5
set protocols bgp group rr neighbor 10.0.0.6
set protocols bgp log-updown

routing-options autonomous-system 1
```

On `vmx15`:

```
set protocols bgp group rr type internal
set protocols bgp group rr local-address 10.0.0.15
set protocols bgp group rr family inet-vpn unicast
set protocols bgp group rr authentication-key "$9$SKwe87-dsoJDwYP5zF/9KMW"
set protocols bgp group rr cluster 0.0.0.1
set protocols bgp group rr neighbor 10.0.1.1
set protocols bgp group rr neighbor 10.0.1.2
set protocols bgp group rr neighbor 10.0.0.5
set protocols bgp group rr neighbor 10.0.0.6
set protocols bgp log-updown

routing-options autonomous-system 1
```

We name the BGP group `rr` and specify the neighbors configured under it will be IBGP neighbors by using `type internal`. Using `local-address` we source the session from the loopback address 

The `family inet-vpn unicast` ensures that the BGP session will be signaled with the proper address family. The `authentication-key` knob authenticates the BGP session.

The `cluster 0.0.0.1` statement is what turns a Juniper router into a route reflector. 

The `log-updown` statement is to have the system log BGP session state changes and finally, we notice that the autonomous system itself is configured under the routing-options.

Now on to the Juniper PE configurations. 

On `vmx5`:

```
set protocols bgp group rr-client type internal
set protocols bgp group rr-client local-address 10.0.0.5
set protocols bgp group rr-client family inet-vpn unicast
set protocols bgp group rr-client authentication-key "$9$SKwe87-dsoJDwYP5zF/9KMW"
set protocols bgp group rr-client neighbor 10.0.0.15
set protocols bgp group rr-client neighbor 10.0.0.14
set protocols bgp log-updown

routing-options autonomous-system 1
```

On `vmx6`:

```
set protocols bgp group rr-client type internal
set protocols bgp group rr-client local-address 10.0.0.6
set protocols bgp group rr-client family inet-vpn unicast
set protocols bgp group rr-client authentication-key "$9$SKwe87-dsoJDwYP5zF/9KMW"
set protocols bgp group rr-client neighbor 10.0.0.15
set protocols bgp group rr-client neighbor 10.0.0.14
set protocols bgp log-updown

routing-options autonomous-system 1
```

Except for the `cluster` statement, the configuration is pretty much the same.

On to the Cisco PE configurations. On `ios_xr_1`, we configure the following:

```
router bgp 1
 bgp router-id 10.0.1.1
 address-family vpnv4 unicast
 !
 neighbor-group rr-client
  remote-as 1
  password encrypted 08324D421D485744
  update-source Loopback0
  address-family vpnv4 unicast
   soft-reconfiguration inbound always
  !
 !
 neighbor 10.0.0.14
  use neighbor-group rr-client
 !
 neighbor 10.0.0.15
  use neighbor-group rr-client
 !
!
```

On `ios_xr_2`:

```
router bgp 1
 bgp router-id 10.0.1.2
 address-family vpnv4 unicast
 !
 neighbor-group rr-client
  remote-as 1
  password encrypted 08324D421D485744
  update-source Loopback0
  address-family vpnv4 unicast
   soft-reconfiguration inbound always
  !
 !
 neighbor 10.0.0.14
  use neighbor-group rr-client
 !
 neighbor 10.0.0.15
  use neighbor-group rr-client
 !
!
```

To verify that all the BGP sessions properly formed, the easiest thing is to check out the route reflectors.

First we check `vmx14`:

<pre>
salt@vmx14> <b>show bgp summary</b>
Threading mode: BGP I/O
Groups: 1 Peers: 4 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       0          0          0          0          0          0
inet6.0              
                       0          0          0          0          0          0
bgp.l3vpn.0          
                       8          8          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.5                  1          4          5       0       4           2 Establ
  bgp.l3vpn.0: 2/2/2/0
10.0.0.6                  1          4          5       0       4           2 Establ
  bgp.l3vpn.0: 2/2/2/0
10.0.1.1                  1          4          4       0       1           8 Establ
  bgp.l3vpn.0: 2/2/2/0
10.0.1.2                  1          4          4       0       1          11 Establ
</pre>

Then we check `vmx15`:

<pre>
salt@vmx15> <b>show bgp summary</b>
Threading mode: BGP I/O
Groups: 1 Peers: 4 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0               
                       0          0          0          0          0          0
inet6.0              
                       0          0          0          0          0          0
bgp.l3vpn.0          
                       8          8          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.5                  1          4          5       0       2          20 Establ
  bgp.l3vpn.0: 2/2/2/0
10.0.0.6                  1          4          5       0       2          20 Establ
  bgp.l3vpn.0: 2/2/2/0
10.0.1.1                  1          4          4       0       1          22 Establ
  bgp.l3vpn.0: 2/2/2/0
10.0.1.2                  1          4          4       0       1          24 Establ
</pre>

After verifying that all sessions are established, we can start verifying that the BGP sessions were properly configured. Since the rest of the BGP session verification will be the same, we will only verify the session between `vmx15` and `ios_xr_1`

First, we check `vmx15`:

<pre>
salt@vmx15> <b>show bgp neighbor 10.0.1.1</b> 
Peer: <b>10.0.1.1+35276 AS 1</b>      Local: <b>10.0.0.15+179 AS 1</b>
  Group: rr                    Routing-Instance: master
  Forwarding routing-instance: master  
  Type: <b>Internal</b>    State: <b>Established</b>  (route reflector client)Flags: &lt;Sync&gt;
  Last State: OpenConfirm   Last Event: RecvKeepAlive
  Last Error: Cease
  Options: &lt;Preference LocalAddress AuthKey LogUpDown Cluster AddressFamily Rib-group Refresh&gt;
  <b>Authentication key is configured</b>
  Address families configured: <b>inet-vpn-unicast</b>
  Local Address: 10.0.0.15 Holdtime: 90 Preference: 170
  Number of flaps: 1
  Last flap event: Stop
  Error: 'Cease' Sent: 1 Recv: 0
  Peer ID: 10.0.1.1        Local ID: 10.0.0.15         Active Holdtime: 90
  Keepalive Interval: 30         Group index: 0    Peer index: 0    SNMP index: 0     
  I/O Session Thread: bgpio-0 State: Enabled
  BFD: disabled, down
  <b>NLRI for restart configured on peer: inet-vpn-unicast</b>
  <b>NLRI advertised by peer: inet-vpn-unicast</b>
  <b>NLRI for this session: inet-vpn-unicast</b>
  Peer supports Refresh capability (2)
  Stale routes from peer are kept for: 300
  Peer does not support Restarter functionality
  Peer does not support Receiver functionality
  Peer does not support LLGR Restarter or Receiver functionality
  Peer supports 4 byte AS extension (peer-as 1)
  Peer does not support Addpath
  NLRI that peer supports extended nexthop encoding for: inet-unicast inet-multicast inet-vpn-unicast
  Table bgp.l3vpn.0 Bit: 40000
    RIB State: BGP restart is complete
    RIB State: VPN restart is complete
    Send state: in sync
    Active prefixes:              2
    Received prefixes:            2
    Accepted prefixes:            2
    Suppressed due to damping:    0
    Advertised prefixes:          6
  Last traffic (seconds): Received 15   Sent 20   Checked 380 
  Input messages:  Total 16     Updates 2       Refreshes 0     Octets 459
  Output messages: Total 17     Updates 3       Refreshes 0     Octets 618
  Output Queue[3]: 0            (bgp.l3vpn.0, inet-vpn-unicast)

</pre>

This output reveals all we need to know. It displays the source and destination IP for the BGP session as well as the AS that is configured on both routers. Additionally, we can see the state of the BGP session as well as the configuration options for this session. We see that the session is authenticated and we can see that the BGP session supports the `inet-vpn-unicast` address family.


Let's move to `ios_xr_1` and check the session from there:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show bgp neighbors 10.0.0.15</b>

BGP neighbor is 10.0.0.15
 <b>Remote AS 1, local AS 1, internal link</b>
 <b>Remote router ID 10.0.0.15</b>
  <b>BGP state = Established, up for 14:07:55</b>
  NSR State: None
  Last read 00:00:02, Last read before reset 14:08:09
  Hold time is 90, keepalive interval is 30 seconds
  Configured hold time: 180, keepalive: 60, min acceptable hold time: 3
  Last write 00:00:17, attempted 19, written 19
  Second last write 00:00:47, attempted 19, written 19
  Last write before reset 14:08:36, attempted 19, written 19
  Second last write before reset 14:09:06, attempted 19, written 19
  Last write pulse rcvd  Aug 21 03:47:03.536 last full not set pulse count 3879
  Last write pulse rcvd before reset 14:08:22
  Socket not armed for io, armed for read, armed for write
  Last write thread event before reset 14:08:22, second last 14:08:23
  Last KA expiry before reset 14:08:36, second last 14:09:06
  Last KA error before reset 00:00:00, KA not sent 00:00:00
  Last KA start before reset 14:08:36, second last 14:09:06
  Precedence: internet
  Non-stop routing is enabled
  Multi-protocol capability received
  Neighbor capabilities:
    Route refresh: advertised (old + new) and received (old + new)
    Graceful Restart (GR Awareness): received
    4-byte AS: advertised and received
    <b>Address family VPNv4 Unicast: advertised and received</b>
  Received 2074 messages, 1 notifications, 0 in queue
  Sent 1838 messages, 1 notifications, 0 in queue
  Minimum time between advertisement runs is 0 secs
  Inbound message logging enabled, 3 messages buffered
  Outbound message logging enabled, 3 messages buffered

 For Address Family: VPNv4 Unicast
  BGP neighbor version 846
  Update group: 0.1 Filter-group: 0.2  No Refresh request being processed
  Inbound soft reconfiguration allowed (override route-refresh)
    Extended Nexthop Encoding: advertised
  Route refresh request: received 0, sent 0
  6 accepted prefixes, 0 are bestpaths
  Exact no. of prefixes denied : 0.
  Cumulative no. of prefixes denied: 0. 
  Prefix advertised 2, suppressed 0, withdrawn 0
  Maximum prefixes allowed 2097152
  Threshold for warning message 75%, restart interval 0 min
  AIGP is enabled
  An EoR was received during read-only mode
  Last ack version 846, Last synced ack version 0
  Outstanding version objects: current 0, max 1, refresh 0
  Additional-paths operation: None
  Send Multicast Attributes

  Connections established 8; dropped 7
  Local host: 10.0.1.1, Local port: 179, IF Handle: 0x00000000
  Foreign host: 10.0.0.15, Foreign port: 51470
  Last reset 14:08:09, due to BGP Notification received: peer unconfigured
  Time since last notification sent to neighbor: 14:15:05
  Error Code: configuration change
  Notification data sent:
    None
  Time since last notification received from neighbor: 14:08:09
  Error Code: peer unconfigured
  Notification data received:
    None
</pre>   

Save for the fact that the session is authenticated, we can see most of the same information in this output.


<br>

Configuring and verifying the MPLS L3VPN
========================================

The MPLS L3VPN we will configure will be a very basic one:

![juniper and ios xr vpn topology](/img/vpn_topology_juniper_ios_xr.png "vpn topology")

 On every PE, we will configure a vrf called `cust-1`. We will place 1 interface inside the VRF and we will configure a single static route towards the customer device. We will cover the example configuration on `ios_xr_1` and on `vmx5`. 


Let's check out the configuration on `ios_xr_1` first. We start by configuring the vrf:

```
vrf cust-1
 rd 1:1
 address-family ipv4 unicast
  import route-target
   1:1
  !
  export route-target
   1:1
  !
 !
!
```

Next, we configure the interface and place that interface in the vrf:

```
interface GigabitEthernet0/0/0/3.2002
 vrf cust-1
 ipv4 address 10.0.0.9 255.255.255.252
 encapsulation dot1q 2002
```

After this, we configure a static route for the vrf under `router static`:

```
router static
 vrf cust-1
  address-family ipv4 unicast
   192.168.1.1/32 10.0.0.10
  !
 !
!
```

Finally, we need to configure BGP to advertise the connected routes and the static for this vrf:

```
router bgp 1
  vrf cust-1
  address-family ipv4 unicast
   label mode per-vrf
   redistribute connected
   redistribute static
  !
 !
!
```

You'll also notice the `label mode per-vrf`. This is not really necessary, but I configured it since I will also use per vrf label allocation on the Juniper.

Now we move to the Juniper configuration on `vmx5`. The configuration components are the same, but everything is organized a bit differently:

```
set interfaces ge-0/0/1 unit 2000 vlan-id 2000
set interfaces ge-0/0/1 unit 2000 family inet address 10.0.0.1/30

set routing-instances cust-1 instance-type vrf
set routing-instances cust-1 interface ge-0/0/1.2000
set routing-instances cust-1 route-distinguisher 1:1
set routing-instances cust-1 vrf-target target:1:1
set routing-instances cust-1 vrf-table-label
set routing-instances cust-1 routing-options static route 192.168.1.3/32 next-hop 10.0.0.2
```

The first thing you'll notice is that the interface configuration is no different from one that is not part of any vrf. After the interface, there is the `routing-instance` configuration. Basically, on a Juniper, everything that has to do with the routing-instance is configured there. From routing-protocols to RDs and RTs.

In our example, we start out specifying what type of vrf it is by specifying the `instance-type`. For MPLS L3VPN, we define the instance as a `vrf`.

Next, we define the interfaces that should be part of the vrf. This is followed by the route-target and route-distinguisher. Through the use of `vrf-target`, we tell the device to export and import the route-target community that is referenced. 

After this is `vrf-table-label`, which ensures the creation of a logical internal interface that enables per vrf label allocation and simplified configuration.

The last configuration item is the static route, which is configured under the routing-options in the routing-instance. 

Now that we have configured the vrf everywhere, we can start our verification. First we verify connectivity from a test device I have connected to the vrf on `ios_xr_1`:

<pre>
salt@test-vmx> <b>ping routing-instance c1-1 rapid source 192.168.1.1 192.168.1.2</b>    
PING 192.168.1.2 (192.168.1.2): 56 data bytes
!!!!!
--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.715/1.964/2.244/0.229 ms

salt@test-vmx> <b>ping routing-instance c1-1 rapid source 192.168.1.1 192.168.1.3</b>  
PING 192.168.1.3 (192.168.1.3): 56 data bytes
!!!!!
--- 192.168.1.3 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.084/3.295/3.457/0.138 ms

salt@test-vmx> <b>ping routing-instance c1-1 rapid source 192.168.1.1 192.168.1.4</b>
PING 192.168.1.4 (192.168.1.4): 56 data bytes
!!!!!
--- 192.168.1.4 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.004/3.490/5.213/0.863 ms
</pre>

So we can exchange ICMP. But there is a lot more to check. Let's check the routing tables on `ios_xr_1` and `vmx5` first:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show route vrf cust-1</b>

Codes: C - connected, S - static, R - RIP, B - BGP, (>) - Diversion path
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2, E - EGP
       i - ISIS, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, su - IS-IS summary null, * - candidate default
       U - per-user static route, o - ODR, L - local, G  - DAGR, l - LISP
       A - access/subscriber, a - Application route
       M - mobile route, r - RPL, t - Traffic Engineering, (!) - FRR Backup path

Gateway of last resort is not set

B    10.0.0.0/30 [200/0] via 10.0.0.5 (nexthop in vrf default), 00:06:36
B    10.0.0.4/30 [200/0] via 10.0.0.6 (nexthop in vrf default), 00:06:36
C    10.0.0.8/30 is directly connected, 00:07:22, GigabitEthernet0/0/0/3.2002
L    10.0.0.9/32 is directly connected, 00:07:22, GigabitEthernet0/0/0/3.2002
B    10.0.0.12/30 [200/0] via 10.0.1.2 (nexthop in vrf default), 00:05:05
S    192.168.1.1/32 [1/0] via 10.0.0.10, 00:07:22
B    192.168.1.2/32 [200/0] via 10.0.1.2 (nexthop in vrf default), 00:05:05
B    192.168.1.3/32 [200/0] via 10.0.0.5 (nexthop in vrf default), 00:06:36
B    192.168.1.4/32 [200/0] via 10.0.0.6 (nexthop in vrf default), 00:06:36
</pre>

Now on the Juniper:

<pre>
salt@vmx5> <b>show route table cust-1</b>

cust-1.inet.0: 9 destinations, 15 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.0/30        *[Direct/0] 5d 12:57:01
                    >  via ge-0/0/1.2000
10.0.0.1/32        *[Local/0] 5d 12:57:01
                       Local via ge-0/0/1.2000
10.0.0.4/30        *[BGP/170] 00:55:09, localpref 100, from 10.0.0.14
                      AS path: I, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
                    [BGP/170] 00:55:09, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
10.0.0.8/30        *[BGP/170] 00:06:42, MED 0, localpref 100, from 10.0.0.14
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 333(top)
                    [BGP/170] 00:06:42, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 333(top)
10.0.0.12/30       *[BGP/170] 00:05:21, MED 0, localpref 100, from 10.0.0.14
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 334(top)
                    [BGP/170] 00:05:21, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 334(top)
192.168.1.1/32     *[BGP/170] 00:06:42, MED 0, localpref 100, from 10.0.0.14
                      AS path: ?, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 325(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 333(top)
                    [BGP/170] 00:06:42, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 325(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 333(top)
192.168.1.2/32     *[BGP/170] 00:05:21, MED 0, localpref 100, from 10.0.0.14
                      AS path: ?, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 326(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 334(top)
                    [BGP/170] 00:05:21, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 24009, Push 326(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 24009, Push 334(top)
192.168.1.3/32     *[Static/5] 5d 12:57:01
                    >  to 10.0.0.2 via ge-0/0/1.2000
192.168.1.4/32     *[BGP/170] 00:55:09, localpref 100, from 10.0.0.14
                      AS path: I, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
                    [BGP/170] 00:55:09, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                       to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
</pre>                    


On both PE devices, we can see that we have learned all of the static routes. Let's check things in a little more detail now and move on to check the route-advertisements and forwarding entries. 

First, we check the signaling from the Juniper to the Cisco and the forwarding from the Cisco to the Juniper. 

![IOS XR to Juniper forwarding](/img/ios_xr_to_vmx_fwd.png "Juniper and IOS XR")

For this reasons, we start out on the Juniper `vmx5` router. We can check the following to see what information is advertised to the route reflector:

<pre>
salt@vmx5> <b>show route advertising-protocol bgp 10.0.0.15 table cust-1 detail</b>

cust-1.inet.0: 9 destinations, 15 routes (9 active, 0 holddown, 0 hidden)
* 10.0.0.0/30 (1 entry, 1 announced)
 BGP group rr-client type Internal
     Route Distinguisher: 1:1
     VPN Label: 282
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [1] I 
     Communities: target:1:1

* 192.168.1.3/32 (1 entry, 1 announced)
 BGP group rr-client type Internal
     Route Distinguisher: 1:1
     VPN Label: 282
     Nexthop: Self
     Flags: Nexthop Change
     Localpref: 100
     AS path: [1] I 
     Communities: target:1:1
</pre>

When we move to the `ios_xr_1`, we can use the following to see the routes that were advertised by `vmx5`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show  bgp vpnv4 unicast vrf cust-1 192.168.1.3/32</b>
BGP routing table entry for 192.168.1.3/32, Route Distinguisher: 1:1
Versions:
  Process           bRIB/RIB  SendTblVer
  Speaker                 12          12
Last Modified: Aug 26 19:28:18.892 for 00:09:43
Paths: (2 available, best #1)
  Not advertised to any peer
  Path #1: Received by speaker 0
  Not advertised to any peer
  Local, (received & used)
    10.0.0.5 (metric 300) from 10.0.0.14 (10.0.0.5)
      Received Label 282 
      Origin IGP, localpref 100, valid, internal, best, group-best, import-candidate, imported
      Received Path ID 0, Local Path ID 1, version 12
      Extended community: RT:1:1 
      Originator: 10.0.0.5, Cluster list: 0.0.0.1
      Source AFI: VPNv4 Unicast, Source VRF: cust-1, Source Route Distinguisher: 1:1
  Path #2: Received by speaker 0
  Not advertised to any peer
  Local, (received & used)
    10.0.0.5 (metric 300) from 10.0.0.15 (10.0.0.5)
      Received Label 282 
      Origin IGP, localpref 100, valid, internal, import-candidate, imported
      Received Path ID 0, Local Path ID 0, version 0
      Extended community: RT:1:1 
      Originator: 10.0.0.5, Cluster list: 0.0.0.1
      Source AFI: VPNv4 Unicast, Source VRF: cust-1, Source Route Distinguisher: 1:1

RP/0/RP0/CPU0:ios_xr_1#<b>show  bgp vpnv4 unicast vrf cust-1 10.0.0.0/30</b>
BGP routing table entry for 10.0.0.0/30, Route Distinguisher: 1:1
Versions:
  Process           bRIB/RIB  SendTblVer
  Speaker                 10          10
Last Modified: Aug 26 19:28:18.892 for 00:09:57
Paths: (2 available, best #1)
  Not advertised to any peer
  Path #1: Received by speaker 0
  Not advertised to any peer
  Local, (received & used)
    10.0.0.5 (metric 300) from 10.0.0.14 (10.0.0.5)
      Received Label 282 
      Origin IGP, localpref 100, valid, internal, best, group-best, import-candidate, imported
      Received Path ID 0, Local Path ID 1, version 10
      Extended community: RT:1:1 
      Originator: 10.0.0.5, Cluster list: 0.0.0.1
      Source AFI: VPNv4 Unicast, Source VRF: cust-1, Source Route Distinguisher: 1:1
  Path #2: Received by speaker 0
  Not advertised to any peer
  Local, (received & used)
    10.0.0.5 (metric 300) from 10.0.0.15 (10.0.0.5)
      Received Label 282 
      Origin IGP, localpref 100, valid, internal, import-candidate, imported
      Received Path ID 0, Local Path ID 0, version 0
      Extended community: RT:1:1 
      Originator: 10.0.0.5, Cluster list: 0.0.0.1
      Source AFI: VPNv4 Unicast, Source VRF: cust-1, Source Route Distinguisher: 1:1
</pre>

So we received the routes with VPN label `282`. If we wanted to understand what the device will do with this information, we can check the following:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show cef vrf cust-1 192.168.1.3</b>
192.168.1.3/32, version 8, internal 0x5000001 0x0 (ptr 0xdf127ec) [1], 0x0 (0xe0d7b28), 0xa08 (0xe7103a8)
 Updated Aug 26 19:28:18.664
 Prefix Len 32, traffic index 0, precedence n/a, priority 3
   via 10.0.0.5/32, 3 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0xd436d90 0x0]
    recursion-via-/32
    next hop VRF - 'default', table - 0xe0000000
    next hop 10.0.0.5/32 via 24004/0/21
     next hop 10.0.2.0/32 Gi0/0/0/1.10 labels imposed {176 282}
     next hop 10.0.2.4/32 Gi0/0/0/2.12 labels imposed {340 282}
</pre>

The `282` label makes sense, we just saw that one in the BGP advertisement. The `176` and `340` are transport labels towards `vmx5`. We can see that using:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show mpls ldp forwarding 10.0.0.5/32</b>

Codes: 
  - = GR label recovering, (!) = LFA FRR pure backup path 
  {} = Label stack with multi-line output for a routing path
  G = GR, S = Stale, R = Remote LFA FRR backup

Prefix          Label   Label(s)       Outgoing     Next Hop            Flags
                In      Out            Interface                        G S R
--------------- ------- -------------- ------------ ------------------- -----
10.0.0.5/32     24004   176            Gi0/0/0/1.10 10.0.2.0                 
                        340            Gi0/0/0/2.12 10.0.2.4                  
</pre>

Following that label is pretty easy. Let's follow the `176` label. The outgoing interface leads us to `vmx1`, so we hop on over to that router and use the following:

<pre>
salt@vmx1> <b>show route table mpls.0 label 176</b>          

mpls.0: 18 destinations, 18 routes (18 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

176                *[LDP/9] 00:23:18, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Swap 318
</pre>                    

This leads us to `vmx4`:

<pre>
salt@vmx4> <b>show route table mpls.0 label 318</b>

mpls.0: 18 destinations, 18 routes (18 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

318                *[LDP/9] 01:02:12, metric 1
                    >  to 192.168.5.0 via ge-0/0/2.5, Pop      
318(S=0)           *[LDP/9] 01:02:12, metric 1
                    >  to 192.168.5.0 via ge-0/0/2.5, Pop      
</pre>

Prior to sending traffic to `vmx5`, the outer label is swapped to `0`. On `vmx5`,  the router will only have to deal with the VPN label (`282`):

<pre>
salt@vmx5> <b>show route table mpls.0 label 282 </b>

mpls.0: 16 destinations, 16 routes (16 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

282                *[VPN/0] 5d 13:04:17
                    >  via lsi.117440513 (cust-1), Pop     
</pre>

The label is popped and further route lookups are done inside the vrf. Following the path to `192.168.1.3`, we would issue the following:

<pre>
salt@vmx5> <b>show route 192.168.1.3</b>

cust-1.inet.0: 9 destinations, 15 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.3/32     *[Static/5] 00:59:53
                    >  to 10.0.0.2 via ge-0/0/1.2000
</pre>        

Let's go the other way around and verify the routing information advertised by `ios_xr_1`, check what is received on `vmx5` and then finish up tracing the forwarding path from `vmx5` to `ios_xr_1`.

![Juniper to IOS XR forwarding](/img/vmx_to_ios_xr_fwd.png "Juniper and IOS XR")

First, we check the route advertisement from `ios_xr_1` by issuing the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show bgp vpnv4 unicast vrf cust-1 advertised neighbor 10.0.0.15</b>
Route Distinguisher: 1:1
10.0.0.8/30 is advertised to 10.0.0.15
  Path info:
    neighbor: Local           neighbor router id: 10.0.1.1
    valid  redistributed  best  import-candidate  
Received Path ID 0, Local Path ID 1, version 17
  Attributes after inbound policy was applied:
    next hop: 0.0.0.0
    MET ORG AS EXTCOMM 
    origin: incomplete  metric: 0  
    aspath: 
    extended community: RT:1:1 
  Attributes after outbound policy was applied:
    next hop: 10.0.1.1
    MET ORG AS EXTCOMM 
    origin: incomplete  metric: 0  
    aspath: 
    extended community: RT:1:1 

Route Distinguisher: 1:1
192.168.1.1/32 is advertised to 10.0.0.15
  Path info:
    neighbor: Local           neighbor router id: 10.0.1.1
    valid  redistributed  best  import-candidate  
Received Path ID 0, Local Path ID 1, version 18
  Attributes after inbound policy was applied:
    next hop: 10.0.0.10
    MET ORG AS EXTCOMM 
    origin: incomplete  metric: 0  
    aspath: 
    extended community: RT:1:1 
  Attributes after outbound policy was applied:
    next hop: 10.0.1.1
    MET ORG AS EXTCOMM 
    origin: incomplete  metric: 0  
    aspath: 
    extended community: RT:1:1 
</pre>

To see the local label, we can use the following:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show mpls forwarding vrf cust-1</b> 
Local  Outgoing    Prefix             Outgoing     Next Hop        Bytes       
Label  Label       or ID              Interface                    Switched    
------ ----------- ------------------ ------------ --------------- ------------
24009  Aggregate   cust-1: Per-VRF Aggr[V]   \
                                      cust-1                       1260  
</pre>

To check what routing information is received on the Juniper device, we can issue the following command on `vmx5`:

<pre>
salt@vmx5> <b>show route receive-protocol bgp 10.0.0.15 table cust-1 next-hop 10.0.1.1 detail</b>

cust-1.inet.0: 9 destinations, 15 routes (9 active, 0 holddown, 0 hidden)
  10.0.0.8/30 (2 entries, 1 announced)
     Import Accepted
     Route Distinguisher: 1:1
     VPN Label: 24009
     Nexthop: 10.0.1.1
     MED: 0
     Localpref: 100
     AS path: ?  (Originator)
     Cluster list:  0.0.0.1
     Originator ID: 10.0.1.1
     Communities: target:1:1

  192.168.1.1/32 (2 entries, 1 announced)
     Import Accepted
     Route Distinguisher: 1:1
     VPN Label: 24009
     Nexthop: 10.0.1.1
     MED: 0
     Localpref: 100
     AS path: ?  (Originator)
     Cluster list:  0.0.0.1
     Originator ID: 10.0.1.1
     Communities: target:1:1
</pre>

The corresponding forwarding entry can be checked like this:

<pre>
salt@vmx5> <b>show route forwarding-table vpn cust-1 destination 192.168.1.1</b>
Logical system: r5
Routing table: cust-1.inet
Internet:
Enabled protocols: Bridging, All VLANs, 
Destination        Type RtRef Next hop           Type Index    NhRef Netif
192.168.1.1/32     user     0                    indr  1048604     3
                                                 ulst  1048602     2
                              192.168.5.1       Push 24009, Push 325(top)     1690     2 ge-0/0/1.5
                              192.168.7.1       Push 24009, Push 333(top)     1691     2 ge-0/0/1.7
</pre>

We just saw that the VPN label, `24009`, was learned through BGP. The top labels, `325` and `333`, are the transport labels. We can see this when we check the routing information to the `ios_xr_1` device (which the VPN routes will resolve to):

<pre>
salt@vmx5> <b>show route 10.0.1.1 table inet.3</b>

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.1/32        *[LDP/9] 00:18:20, metric 301
                       to 192.168.5.1 via ge-0/0/1.5, Push 325
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 333
</pre>

Let's follow label `325`, which is send to `vmx4`:

<pre>
salt@vmx4> <b>show route table mpls.0 label 325</b>

mpls.0: 18 destinations, 18 routes (18 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

325                *[LDP/9] 00:18:49, metric 1
                    >  to 192.168.4.0 via ge-0/0/2.4, Swap 181

</pre>

This is forwarded to `vmx1`:

<pre>
salt@vmx1> <b>show route table mpls.0 label 181</b>

mpls.0: 18 destinations, 18 routes (18 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

181                *[LDP/9] 00:19:07, metric 1
                    >  to 10.0.2.1 via ge-0/0/3.10, Pop      
181(S=0)           *[LDP/9] 00:19:07, metric 1
                    >  to 10.0.2.1 via ge-0/0/3.10, Pop  
</pre>

This will leave the Cisco to deal with the explicit null label and the VPN label we saw earlier (`24009`).


Closing thoughts
================

These are the basics to get a VRF going between Juniper and Cisco IOS XR. There are differences in configuration constructs, but coming from a Juniper background and having done some IOS a long time ago, I found the IOS XR not too difficult to understand or work with. 















    