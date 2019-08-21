---
layout: post
title: MPLS L3VPN between Juniper MX and Cisco IOS XR
image: /img/juniper_logo.jpg
---

The topology

The setup:
- OSPF IGP: p2p links, authentication, BFD, load-balaning
- MPLS LDP: explicit null, session authenticaiton, ldp sync
- BGP: 2 RRs, VPNv4 AF, authenticaiton
- VRF with static routes
- VRF dynamic routing
- adding v6 everywhere

<br>

Interface configuration
=======================

The interface, OSPF and LDP configuration is going to be the same on every device. For that reason, the example configuration will include the configuration and verification steps on `ios_xr_1` and `vmx1`. 

Configuring the interfaces on the `IOS XR`:

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

Configuring the interface on the `Juniper vMX`:

```
set interfaces ge-0/0/3 flexible-vlan-tagging
set interfaces ge-0/0/3 encapsulation flexible-ethernet-services
set interfaces ge-0/0/3 unit 10 description ios_xr_1
set interfaces ge-0/0/3 unit 10 vlan-id 10
set interfaces ge-0/0/3 unit 10 family inet address 10.0.2.0/31

set interfaces lo0 unit 1 family inet address 10.0.0.1/32
```

Apart from the obvious difference in syntax, there are two items here to consider. The Cisco interfaces need to be enabled before they can be used. After working with Juniper for so long, I actually forgot about this when I was trying to get the IOS XR going on KVM.

On the Juniper device, we need to ready the interface to allow different encapsulation on the main interface. Because `flexible-vlan-tagging` and `encapsulation flexible-ethernet-services` will enable most of the scenario's, I generally default to using that.

To verify the interface status on the `IOS XR`, we issue to the following command:

```
RP/0/RP0/CPU0:ios_xr_1#show interface GigabitEthernet0/0/0/1.10      
Mon Aug 19 09:20:26.399 UTC
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
```     


Same thing on the `Juniper`:

```
salt@vmx01:r1> show interfaces ge-0/0/3.10    
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
```    

To wrap up interfaces configuration, we verify that we are able to ping accross the interface:

```
salt@vmx01:r1> ping 10.0.2.1 rapid 
PING 10.0.2.1 (10.0.2.1): 56 data bytes
!!!!!
--- 10.0.2.1 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 3.581/4.794/5.613/0.668 ms
```



Configuring and verifying the IGP
=================================

We are going with several options common to most networks I worked on. What will be included in the IGP configuration is the following:
- OSPF area 0 interface configuration
- calculate the link cost factoring in future 100G links
- authentication
- BFD
- load-balancing

The following is the `IOS XR` configuration:

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


The equivalant configuration on the `Juniper` is the following:

```
set protocols ospf reference-bandwidth 100g
set protocols ospf area 0.0.0.0 interface lo0.1 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 authentication md5 1 key "$9$HmQn/9p1RStuWL7NbwP5T"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 bfd-liveness-detection minimum-interval 100
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 bfd-liveness-detection multiplier 5
```

Since load balancing is not something that is automatically enabled on Juniper devices, we need to add the following to `vmx01`:

```
set policy-options policy-statement lbpp term 1 then load-balance per-packet
set routing-options forwarding-table export lbpp
```

Do not let the `load-balance per-packet` name fool you. This action will have the Juniper load balance traffic per flow.

After the configuration is applied on both devices, we can start our verification. 

First, we check the OSPF neighbor status on both sides.

On the Cisco, we check the following:

```
RP/0/RP0/CPU0:ios_xr_1#show ospf neighbor 10.0.0.1 
Sat Aug 17 19:40:19.729 UTC

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
```

The neighbor state is `FULL` and the the BFD session that provides us with fast failure detection is also up. Other details can be verified by issuing the `show ospf 1 interface` command, like so:

```
RP/0/RP0/CPU0:ios_xr_1#show ospf 1 interface GigabitEthernet0/0/0/1.10             
Sat Aug 17 19:59:51.843 UTC

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
```

Here we see that the interface is of the type `POINT_TO_POINT`. We can also see the BFD settings here. And lastly, we see what authentication is configured.

In case we really wanted to know about all the details on the BFD session, we should turn to the following command:
```
RP/0/RP0/CPU0:ios_xr_1#show bfd session interface GigabitEthernet0/0/0/1.10 detail 
Mon Aug 19 14:20:59.517 UTC
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
```  


Let's verify the same things on the Juniper side and start off checking the OSPF neighbor state:

```
salt@vmx01:r1> show ospf neighbor 10.0.1.1 extensive        
Address          Interface              State     ID               Pri  Dead
10.0.2.1         ge-0/0/3.10            Full      10.0.1.1           1    37
  Area 0.0.0.0, opt 0x52, DR 0.0.0.0, BDR 0.0.0.0
  Up 23:08:38, adjacent 23:08:38
  Topology default (ID 0) -> Bidirectional
```

```
salt@vmx01:r1> show ospf interface ge-0/0/3.10 extensive    
Interface           State   Area            DR ID           BDR ID          Nbrs
ge-0/0/3.10         PtToPt  0.0.0.0         0.0.0.0         0.0.0.0            1
  Type: P2P, Address: 10.0.2.0, Mask: 255.255.255.254, MTU: 1500, Cost: 1
  Adj count: 1
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  Auth type: MD5, Active key ID: 1, Start time: 1970 Jan  1 00:00:00 UTC
  Protection type: None
  Topology default (ID 0) -> <b>Cost: 100</b>

salt@vmx01:r1> show bfd session extensive                   
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
```  

Both routers should have an OSPF route towards their neighboring loopback IP now. We can verify things on the IOS XR like so:

```
RP/0/RP0/CPU0:ios_xr_1#show route 10.0.0.1
Tue Aug 20 08:28:48.387 UTC

Routing entry for 10.0.0.1/32
  Known via "ospf 1", distance 110, metric 100, type intra area
  Installed Aug 20 08:23:57.320 for 00:04:51
  Routing Descriptor Blocks
    10.0.2.0, from 10.0.0.1, via GigabitEthernet0/0/0/1.10
      Route metric is 100
  No advertising protos.  
```

On the Juniper:

```
salt@vmx01:r1> show route 10.0.1.1    

inet.0: 30 destinations, 30 routes (30 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.1.1/32        *[OSPF/10] 00:04:24, metric 101
                    >  to 10.0.2.1 via ge-0/0/3.10
```                    

Notice the difference in cost. Both Juniper and Cisco assigned a cost of 100 to the link, but the cost assigned to the loopback interface differs. On the Cisco:

```
RP/0/RP0/CPU0:ios_xr_1#show ospf interface loopback 0
Tue Aug 20 08:58:29.481 UTC

Loopback0 is up, line protocol is up 
  Internet Address 10.0.1.1/32, Area 0
  Label stack Primary label 0 Backup label 0 SRTE label 0
  Process ID 1, Router ID 10.0.1.1, Network Type LOOPBACK, <b>Cost: 1</b>
  Loopback interface is treated as a stub Host
```

On the Juniper:
```
salt@vmx01:r1> show ospf interface lo0.1 detail 
Interface           State   Area            DR ID           BDR ID          Nbrs
lo0.1               DRother 0.0.0.0         0.0.0.0         0.0.0.0            0
  Type: LAN, Address: 10.0.0.1, Mask: 255.255.255.255, MTU: 65535, <b>Cost: 0</b>
  Adj count: 0, Passive
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  Auth type: None
  Protection type: None
  Topology default (ID 0) -> Passive, Cost: 0
```


Last thing is to verify load-balancing. On the Cisco, we check the route towards the other IOS XR PE:
```
RP/0/RP0/CPU0:ios_xr_1#show route 10.0.1.2
Tue Aug 20 08:52:31.483 UTC

Routing entry for 10.0.1.2/32
  Known via "ospf 1", distance 110, metric 201, type intra area
  Installed Aug 20 08:52:01.911 for 00:00:29
  Routing Descriptor Blocks
    10.0.2.0, from 10.0.1.2, via GigabitEthernet0/0/0/1.10
      Route metric is 201
    10.0.2.4, from 10.0.1.2, via GigabitEthernet0/0/0/2.12
      Route metric is 201
  No advertising protos.
```  

On the Juniper, we check the route towards R3:

```

salt@vmx01:r1> show route 10.0.0.3                                 

inet.0: 30 destinations, 30 routes (30 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.0.3/32        *[OSPF/10] 00:00:12, metric 200
                       to 192.168.4.1 via ge-0/0/1.4
                    >  to 192.168.1.1 via ge-0/0/1.1

salt@vmx01:r1> show route forwarding-table destination 10.0.0.3    
Logical system: r1
Routing table: default.inet
Internet:
Enabled protocols: Bridging, Dual VLAN, 
Destination        Type RtRef Next hop           Type Index    NhRef Netif
10.0.0.3/32        user     0                    ulst  1048587     4
                              192.168.4.1        ucst      843    12 ge-0/0/1.4
                              192.168.1.1        ucst      842     8 ge-0/0/1.1
```



<br>


Configuring and verifying LDP
=============================

To get any MPLS L3VPN going, we require an MPLS network. We are going to take the easy pick and settle for LDP. The LDP configuration will be straightforward and include the following:
- authentication
- advertisement of an explicit null label
- advertisement of the loopback IP only
- FEC deaggregation
- LDP synchronization

Additionally, we want to ensure that the IGP metrics are reflected in the LDP routes.

The LDP configuration is the same on every router, so again, we focus on the configuration between ios_xr_1 and vMX1. 


First, the Cisco MPLS configuration:

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
    advertise
     explicit-null
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

Now the Juniper configuration:

```
set interfaces ge-0/0/3 unit 10 family mpls

set protocols mpls interface ge-0/0/3.10

set protocols ldp track-igp-metric
set protocols ldp deaggregate
set protocols ldp explicit-null
set protocols ldp interface ge-0/0/3.10
set protocols ldp session-group 0.0.0.0/0 authentication-key "$9$.539AtOEcl0BX7dVY2TzF"

set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 ldp-synchronization
```

We notice a few things. First of all, specifying the interfaces under LDP is enough for IOS XR. For the Juniper, to enable the interface for OSPF, configuring it under LDP is not enough. We also need to enable the interfaces for MPLS processing by specifying the MPLS address family under the interface configuration and under  `protocols mpls`.

The other thing we notice is that the defaults for Juniper and Cisco are slighlty different. 

For Juniper, we configured the following options:
- `track-igp-metric`: this will copy the IGP metric into the `inet.3` table. Normally, Juniper will default all LDP routes to 1 _ascii shrug_
- `deaggregate`: this will deaggregate the FECs and will cause each prefix to be bound to a separate label. Not the default on Juniper, though it is recommended to configure it by Juniper.

For Cisco, we configured the following options:
- `mpls ldp address-family ipv4 label local allocate for host-routes`: this causes for the IOS XR router to advertise a label for the lo0 interface only. The default is to advertise a label for every interface.

The other configuration items are pretty similar. Though different in syntac, on both Juniper and Cisco we configure the device to:
- advertise the explicit null label
- enable LDP synchronization by specifying that under the OSPF stanza
- authenticate all LDP sessions through a single configuration command

Both Cisco ass well as Juniper offer ways to inject into LDP whatever routes you want, both offer BFD protection and many more configuration options, but I do not want to go off to far into the weeds here. Instead, we will verify what we have configured so far.

First, we check the LDP adjacency state on Cisco:

```
RP/0/RP0/CPU0:ios_xr_1#show mpls ldp discovery 
Tue Aug 20 11:56:56.417 UTC

Local LDP Identifier: 10.0.1.1:0
Discovery Sources:
  Interfaces:
    GigabitEthernet0/0/0/1.10 : xmit/recv
      VRF: 'default' (0x60000000)
      LDP Id: 10.0.0.1:0, Transport address: 10.0.0.1
          Hold time: 15 sec (local:15 sec, peer:15 sec)
          Established: Aug 20 11:56:03.380 (00:00:53 ago)

```

Next on Juniper:
```
salt@vmx01:r1> show ldp neighbor                 
Address                             Interface       Label space ID     Hold time
10.0.2.1                            ge-0/0/3.10     10.0.1.1:0           10
```

After forming this adjaceny, an LDP session is established. To verify the LDP session, we issue the following command on the Cisco device:

```
RP/0/RP0/CPU0:ios_xr_1#show mpls ldp neighbor  
Tue Aug 20 11:57:06.842 UTC

Peer LDP Identifier: 10.0.0.1:0
  TCP connection: 10.0.0.1:646 - 10.0.1.1:53366; MD5 on
  Graceful Restart: No
  Session Holdtime: 30 sec
  State: Oper; Msgs sent/rcvd: 19/18; Downstream-Unsolicited
  Up time: 00:00:59
  LDP Discovery Sources:
    IPv4: (1)
      GigabitEthernet0/0/0/1.10
    IPv6: (0)
  Addresses bound to this peer:
    IPv4: (5)
      10.0.2.0       10.0.2.2       192.168.1.0    192.168.4.0    
      192.168.15.0   
    IPv6: (0)

```

And on the Juniper device:
```
salt@vmx01:r1> show ldp session detail      
Address: 10.0.1.1, State: Operational, Connection: Open, Hold time: 25
  Session ID: 10.0.0.1:0--10.0.1.1:0
  Next keepalive in 8 seconds
  Passive, Maximum PDU: 4096, Hold time: 30, Neighbor count: 1
  Neighbor types: discovered
  Keepalive interval: 10, Connect retry interval: 1
  Local address: 10.0.0.1, Remote address: 10.0.1.1
  Up for 00:07:49
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

```

Next we check the LDP synchronization. It was configured under OSPF, and on both Cisco as well as Juniper, that is the place where we verify it as well. First, we check the Cisco:

```
RP/0/RP0/CPU0:ios_xr_1#show ospf interface GigabitEthernet0/0/0/1.10 | include LDP
Tue Aug 20 12:07:51.833 UTC
  LDP Sync Enabled, Sync Status: Achieved
```

Then we check the Juniper:

```
salt@vmx01:r1> show ospf interface ge-0/0/3.10 extensive | match ldp 
  LDP sync state: in sync, for: 00:13:39, reason: LDP session up
```

If all is well, both devices have now signaled an LDP LSP between them. Let's verify this on the Cisco side:

```
RP/0/RP0/CPU0:ios_xr_1#show cef 10.0.0.1
Tue Aug 20 12:19:48.209 UTC
10.0.0.1/32, version 5, internal 0x1000001 0x0 (ptr 0xe1cbd88) [1], 0x0 (0xe3900a8), 0xa28 (0xea28328)
 Updated Aug 20 11:56:54.109 
 remote adjacency to GigabitEthernet0/0/0/1.10
 Prefix Len 32, traffic index 0, precedence n/a, priority 3
   via 10.0.2.0/32, GigabitEthernet0/0/0/1.10, 2 dependencies, weight 0, class 0 [flags 0x0]
    path-idx 0 NHID 0x0 [0xf1412f0 0x0]
    next hop 10.0.2.0/32
    remote adjacency
     local label 24000      labels imposed {ExpNullv4}

RP/0/RP0/CPU0:ios_xr_1#show mpls ldp ipv4 forwarding 10.0.0.1/32
Tue Aug 20 12:20:40.101 UTC

Codes: 
  - = GR label recovering, (!) = LFA FRR pure backup path 
  {} = Label stack with multi-line output for a routing path
  G = GR, S = Stale, R = Remote LFA FRR backup

Prefix          Label   Label(s)       Outgoing     Next Hop            Flags
                In      Out            Interface                        G S R
--------------- ------- -------------- ------------ ------------------- -----
10.0.0.1/32     24000   ExpNullv4      Gi0/0/0/1.10 10.0.2.0                  
```

And now on the Juniper side:

```
salt@vmx01:r1> show route 10.0.1.2/32 

inet.0: 31 destinations, 40 routes (31 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.1.2/32        @[OSPF/10] 00:26:33, metric 101
                    >  to 10.0.2.3 via ge-0/0/3.11
                   #[LDP/9] 00:26:33, metric 1
                    >  to 10.0.2.3 via ge-0/0/3.11, Push 0

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.2/32        *[LDP/9] 00:26:33, metric 1
                    >  to 10.0.2.3 via ge-0/0/3.11, Push 0
```


The LDP configuration is the same for every device in the topology. After enabling the configuration on every device, we can start verifying whether or not we have all the required routes.

We can check this on the Cisco using the following commands:

```
RP/0/RP0/CPU0:ios_xr_1#show mpls ldp ipv4 forwarding
Tue Aug 20 12:42:01.827 UTC

Codes: 
  - = GR label recovering, (!) = LFA FRR pure backup path 
  {} = Label stack with multi-line output for a routing path
  G = GR, S = Stale, R = Remote LFA FRR backup

Prefix          Label   Label(s)       Outgoing     Next Hop            Flags
                In      Out            Interface                        G S R
--------------- ------- -------------- ------------ ------------------- -----
10.0.0.1/32     24000   ExpNullv4      Gi0/0/0/1.10 10.0.2.0                 
10.0.0.2/32     24001   ExpNullv4      Gi0/0/0/2.12 10.0.2.4                 
10.0.0.3/32     24002   92             Gi0/0/0/2.12 10.0.2.4                 
10.0.0.4/32     24003   37             Gi0/0/0/1.10 10.0.2.0                 
10.0.0.5/32     24004   38             Gi0/0/0/1.10 10.0.2.0                 
                        94             Gi0/0/0/2.12 10.0.2.4                 
10.0.0.6/32     24005   39             Gi0/0/0/1.10 10.0.2.0                 
                        95             Gi0/0/0/2.12 10.0.2.4                 
10.0.0.14/32    24006   40             Gi0/0/0/1.10 10.0.2.0                 
10.0.0.15/32    24007   49             Gi0/0/0/1.10 10.0.2.0                 
10.0.1.2/32     24008   62             Gi0/0/0/1.10 10.0.2.0                 
                        243            Gi0/0/0/2.12 10.0.2.4 
```

Next, we check the the same thing on the Juniper:

```
salt@vmx01:r1> show route protocol ldp table inet.3 

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.2/32        *[LDP/9] 00:10:22, metric 1
                    >  to 192.168.1.1 via ge-0/0/1.1, Push 0
10.0.0.3/32        *[LDP/9] 00:10:22, metric 1
                       to 192.168.4.1 via ge-0/0/1.4, Push 64
                    >  to 192.168.1.1 via ge-0/0/1.1, Push 92
10.0.0.4/32        *[LDP/9] 00:10:22, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 0
10.0.0.5/32        *[LDP/9] 00:10:22, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 66
10.0.0.6/32        *[LDP/9] 00:10:22, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 67
10.0.0.14/32       *[LDP/9] 00:10:22, metric 1
                    >  to 192.168.4.1 via ge-0/0/1.4, Push 68
10.0.0.15/32       *[LDP/9] 00:10:22, metric 1
                    >  to 192.168.15.1 via ge-0/0/1.15, Push 0
10.0.1.1/32        *[LDP/9] 00:02:22, metric 1
                    >  to 10.0.2.1 via ge-0/0/3.10, Push 0
10.0.1.2/32        *[LDP/9] 00:10:22, metric 1
                    >  to 10.0.2.3 via ge-0/0/3.11, Push 0
```

Both devices offer plenty of other commands to really get into the nitty gritty but I am going to skip that here. Additional verification of label information will be done when the VPN is setup.



<br>

Configuring and verifying BGP
=============================

We configure BGP on the route reflectors and on the PE routers. Let's examine the configuration on the route reflectors first.

On `r14`:
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

On `r15`:
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

We name the BGP group RR and specify the neighbors configured under it will be IBGP neighbors by using `type internal`. Using `local-address` we source the session from the loopback address 

The `family inet-vpn unicast` ensures that the BGP session will be signaled with the proper address family. The `authentication-key` knob authenticates the BGP session.

The `cluster 0.0.0.1` statement is what turns this router into a route reflector. 

The `log-updown` statement is to have the system log BGP session state changes and finally, we notice that the autonomous system itself is configured under the routing-options.

Now on to the Juniper PE configurations. 

On `r5`:

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

On `r6`:

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

Now, on to the Cisco PE configurations. On `ios_xr_1`, we have the following configuration:

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

On `ios_xr_2`, we have the following configuration:

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

First we check `r14`:

```
salt@vmx01:r14> show bgp summary    
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
```

Then we check `r15`:

```
salt@vmx01:r15> show bgp summary    
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
```

After verifying that all sessions are established, we can start verifying that the BGP sessions were properly configured. As an example, we will verify the session between `r15` and `ios_xr_1`. The rest of the BGP session configuration will be the same. 

First, we check `r15`:

```
salt@vmx01:r15> show bgp neighbor 10.0.1.1 
Peer: <b>10.0.1.1+35276 AS 1</b>      Local: <b>10.0.0.15+179 AS 1</b>
  Group: rr                    Routing-Instance: master
  Forwarding routing-instance: master  
  Type: <b>Internal</b>    State: <b>Established</b>  (route reflector client)Flags: <Sync>
  Last State: OpenConfirm   Last Event: RecvKeepAlive
  Last Error: Cease
  Options: <Preference LocalAddress AuthKey LogUpDown Cluster AddressFamily Rib-group Refresh>
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

```

This output reveals all we need to know. It displays the source and destination IP for the BGP session as well as the AS that is configured on both routers. Additionally, we can see the state of the BGP session as well as the configuration options for this session. We see that the session is authenticated and we can see that the BGP session supports the `inet-vpn-unicast` address family.


Let's move to `ios_xr_1` and check the session from there:

```
RP/0/RP0/CPU0:ios_xr_1#show bgp neighbors 10.0.0.15
Wed Aug 21 03:47:05.585 UTC

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
```    

Save for the fact that the session is authenticated, we can see most of the same information in this output.


<br>

Configuring and verifying the MPLS L3VPN
========================================

.