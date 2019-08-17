---
layout: post
title: MPLS L3VPN between Juniper MX and Cisco IOS XR
image: /img/juniper_logo.jpg
---

The topology

The setup:
- OSPF IGP: p2p links, authentication, BFD
- MPLS LDP: explicit null, session authenticaiton, ldp sync
- BGP: 2 RRs, VPNv4 AF, authenticaiton
- VRF with static routes

<br>

Configuring and verifying the IGP
=================================

Since the OSPF configuration is the same on every device, let's focus in on the configuration that is required to get OSPF running between `ios_xr_1` and `vmx1`. We are going with several standard options, so what will be configured includes the following:
- configuring the OSPF interface as point to point ( less entries in the OSPF database and faster neighborship formation)
- authenticating OSPF
- enabling BFD for fast failure detection


The following is the `IOS XR` configuration:

```
interface Loopback0
 ipv4 address 10.0.1.1 255.255.255.255
 
interface GigabitEthernet0/0/0/1.10
 description vmx1
 ipv4 address 10.0.2.1 255.255.255.254
 encapsulation dot1q 1

router ospf 1
 router-id 10.0.1.1
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


The following is the `Juniper` configuration:

```
set interfaces ge-0/0/3 flexible-vlan-tagging
set interfaces ge-0/0/3 encapsulation flexible-ethernet-services
set interfaces ge-0/0/3 unit 10 description ios_xr_1
set interfaces ge-0/0/3 unit 10 vlan-id 10
set interfaces ge-0/0/3 unit 10 family inet address 10.0.2.0/31
set interfaces ge-0/0/3 unit 10 family mpls

set interfaces lo0 unit 1 family inet address 10.0.0.1/32

set protocols ospf area 0.0.0.0 interface lo0.1 passive
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 interface-type p2p
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 authentication md5 1 key "$9$HmQn/9p1RStuWL7NbwP5T"
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 bfd-liveness-detection minimum-interval 100
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 bfd-liveness-detection multiplier 5
```

After the configuration is applied on both devices, we can start our verification. First, we check the OSPF neighbor status on both sides:

```
RP/0/RP0/CPU0:ios_xr_1#show ospf neighbor 10.0.0.1 
Sat Aug 17 19:40:19.729 UTC

* Indicates MADJ interface
# Indicates Neighbor awaiting BFD session up

Neighbors for OSPF 1

 Neighbor 10.0.0.1, interface address 10.0.2.0
    In the area 0 via interface GigabitEthernet0/0/0/1.10 , BFD enabled, Mode: Default
    Neighbor priority is 128, State is FULL, 7 state changes
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
    Neighbor BFD status: Session up

RP/0/RP0/CPU0:ios_xr_1#show ospf 1 interface GigabitEthernet0/0/0/1.10             
Sat Aug 17 19:59:51.843 UTC

GigabitEthernet0/0/0/1.10 is up, line protocol is up 
  Internet Address 10.0.2.1/31, Area 0
  Label stack Primary label 1 Backup label 3 SRTE label 10
  Process ID 1, Router ID 10.0.1.1, Network Type POINT_TO_POINT, Cost: 1
  Transmit Delay is 1 sec, State POINT_TO_POINT, MTU 1500, MaxPktSz 1500
  Forward reference No, Unnumbered no,  Bandwidth 1000000 
  BFD enabled, BFD interval 100 msec, BFD multiplier 5, Mode: Default
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
  Message digest authentication enabled
    Youngest key id is 1
  Multi-area interface Count is 0
RP/0/RP0/CPU0:ios_xr_1#show bfd session interface GigabitEthernet0/0/0/1.10 
Sat Aug 17 19:59:54.256 UTC
Interface           Dest Addr           Local det time(int*mult)      State     
                                    Echo             Async   H/W   NPU     
------------------- --------------- ---------------- ---------------- ----------
Gi0/0/0/1.10        10.0.2.0        0s(0s*0)         500ms(100ms*5)   UP        
                                                             No    n/a      
```

The verification on the vMX:

```
salt@vmx01:r1> show ospf neighbor 10.0.1.1 extensive        
Address          Interface              State     ID               Pri  Dead
10.0.2.1         ge-0/0/3.10            Full      10.0.1.1           1    37
  Area 0.0.0.0, opt 0x52, DR 0.0.0.0, BDR 0.0.0.0
  Up 23:08:38, adjacent 23:08:38
  Topology default (ID 0) -> Bidirectional

salt@vmx01:r1> show ospf interface ge-0/0/3.10 extensive    
Interface           State   Area            DR ID           BDR ID          Nbrs
ge-0/0/3.10         PtToPt  0.0.0.0         0.0.0.0         0.0.0.0            1
  Type: P2P, Address: 10.0.2.0, Mask: 255.255.255.254, MTU: 1500, Cost: 1
  Adj count: 1
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  Auth type: MD5, Active key ID: 1, Start time: 1970 Jan  1 00:00:00 UTC
  Protection type: None
  Topology default (ID 0) -> Cost: 1

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
Sat Aug 17 20:01:50.817 UTC

Routing entry for 10.0.0.1/32
  Known via "ospf 1", distance 110, metric 1, type intra area
  Installed Aug 16 21:06:19.615 for 22:55:31
  Routing Descriptor Blocks
    10.0.2.0, from 10.0.0.1, via GigabitEthernet0/0/0/1.10
      Route metric is 1
  No advertising protos. 
```

On the Juniper:

```
salt@vmx01:r1> show route 10.0.1.1    

inet.0: 30 destinations, 30 routes (30 active, 0 holddown, 0 hidden)
@ = Routing Use Only, # = Forwarding Use Only
+ = Active Route, - = Last Active, * = Both

10.0.1.1/32        *[OSPF/10] 22:56:38, metric 2
                    >  to 10.0.2.1 via ge-0/0/3.10
```                    

All other interfaces that connect the routers in the topology will have the same OSPF interface configuration.

<br>

Configuring and verifying LDP
=============================

.
set protocols ospf area 0.0.0.0 interface ge-0/0/3.10 ldp-synchronization
<br>

Configuring and verifying BGP
=============================

.

<br>

Configuring and verifying the MPLS L3VPN
========================================

.