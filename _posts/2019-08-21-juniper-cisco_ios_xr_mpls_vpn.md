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