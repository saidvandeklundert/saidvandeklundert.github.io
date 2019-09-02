---
layout: post
title: 6VPE between Juniper MX and Cisco IOS XR
image: /img/cisco_juniper_logo.png
---


Previously, I got an MPLS L3VPN going in this post:
[MPLS L3VPN between Juniper MX and Cisco IOS XR](http://saidvandeklundert.net/2019-08-28-juniper-cisco_ios_xr_mpls_vpn/ ) 

The aim right now is to enable the same VPN for IPv6 using 6VPE.

The steps to enable the topology for 6VPE are the following:
- enable the CE-facing PE interfaces for IPv6
- configure the IPv6 static routes
- ready the VRF configuration for IPv6
- enable the BGP sessions to carry VPN-IPv6 addresses
- enable IPv6 tunneling
- review the complete VPN configuration
- verify the 6VPE VPN

For the PE configuration, the samples I will show are from `ios_xr_1` and `vmx5`.

Enabling the CE-facing PE interfaces for IPv6
=============================================

To enable the CE-facing interface on `ios_xr_1` for IPv6, we add the following:

```
interface GigabitEthernet0/0/0/3.2002
 ipv6 address 2001:db8:1::1/127
```
For `ios_xr_1`, we now have the following interface configuration:

```
interface GigabitEthernet0/0/0/3.2002
 description c1-1
 vrf cust-1
 ipv4 address 10.0.0.9 255.255.255.252
 ipv6 address 2001:db8:1::1/127
 encapsulation dot1q 2002
```


On `vmx5`, we add the following:

```
set interfaces ge-0/0/1 unit 2000 family inet6 address 2001:db8:1::5/127
```

This entire CE-facing interface configuration on `vmx5` is the following now:

```
set interfaces ge-0/0/1 flexible-vlan-tagging
set interfaces ge-0/0/1 encapsulation flexible-ethernet-services
set interfaces ge-0/0/1 unit 2000 vlan-id 2000
set interfaces ge-0/0/1 unit 2000 family inet address 10.0.0.1/30
set interfaces ge-0/0/1 unit 2000 family inet6 address 2001:db8:1::5/127
```


To verify the IOS XR configuration, we issue the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show ipv6 vrf cust-1 interface GigabitEthernet0/0/0/3.2002 brief</b>
GigabitEthernet0/0/0/3.2002 [Up/Up]
    fe80::5046:84ff:fe57:a006                     
    2001:db8:1::1
</pre>

To verify the `vmx` configuration, we issue the following command:

<pre>
salt@vmx01:r5> <b>show interfaces ge-0/0/1.2000 terse</b>
Interface               Admin Link Proto    Local                 Remote
ge-0/0/1.2000           up    up   inet     10.0.0.1/30     
                                   inet6    2001:db8:1::5/127
                                            fe80::5254:7:d07c:df65/64
</pre>


Configuring the static routes
=============================

On `ios_xr_1`, we apply the following configuration:

```
router static
 vrf cust-1
  address-family ipv6 unicast
   2001:db8::1/128 2001:db8:1::
  !
 !
```

On `vmx5`, we configure the following:

```
set routing-instances cust-1 routing-options rib cust-1.inet6.0 static route 2001:db8::3/128 next-hop 2001:db8:1::4
```

When we verify the static routes, we should see something like this on the `ios_xr_1`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show route vrf cust-1 ipv6 2001:db8::1/128</b>

Routing entry for 2001:db8::1/128
  Known via "static", distance 1, metric 0
  Installed Aug 30 07:45:39.948 for 1d01h
  Routing Descriptor Blocks
    2001:db8:1::
      Route metric is 0, Wt is 1
  No advertising protos. 
</pre>

When we check `vmx5`, we do so using the following command:

<pre>
salt@vmx01:r5> <b>show route 2001:db8::3/128 table cust-1</b>

cust-1.inet6.0: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8::3/128    *[Static/5] 1d 01:26:08
                    >  to 2001:db8:1::4 via ge-0/0/1.2000
</pre>


Ready the VRF configuration for IPv6
====================================


On `ios_xr_1`, we add the following configuration to the vrf:

```
vrf cust-1
 address-family ipv6 unicast
  import route-target
   1:1
  !
  export route-target
   1:1
  !
 !
!
```

On `ios_xr_1`, we issue the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show vrf cust-1 detail </b>

VRF cust-1; RD 1:1; VPN ID not set
VRF mode: Regular
Description not set
Interfaces:
  GigabitEthernet0/0/0/3.2002
Address family IPV4 Unicast
  Import VPN route-target communities:
    RT:1:1
  Export VPN route-target communities:
    RT:1:1
  No import route policy
  No export route policy
Address family IPV6 Unicast
  Import VPN route-target communities:
    RT:1:1
  Export VPN route-target communities:
    RT:1:1
  No import route policy
  No export route policy
</pre>

We see the import and export now also applies to IPv6.

The `vmx` requires no such configuration. After enabling the interface in the VRF for IPv6 and setting a static route for the vrf, we can see the following:

<pre>
salt@vmx01:r5> <b>show route instance cust-1 extensive</b>
cust-1:
  Router ID: 10.0.0.1
  Type: vrf               State: Active        
  Interfaces:
    ge-0/0/1.2000
    lsi.117440513
  Route-distinguisher: 1:1
  Vrf-import: [ __vrf-import-cust-1-internal__ ]
  Vrf-export: [ __vrf-export-cust-1-internal__ ]
  Vrf-import-target: [ target:1:1 ]
  Vrf-export-target: [ target:1:1 ]
  Fast-reroute-priority: low
  Tables:
    cust-1.inet.0          : 15 routes (9 active, 0 holddown, 0 hidden)
    cust-1.iso.0           : 0 routes (0 active, 0 holddown, 0 hidden)
    cust-1.inet6.0         : 5 routes (5 active, 0 holddown, 0 hidden)
    cust-1.mdt.0           : 0 routes (0 active, 0 holddown, 0 hidden
</pre>



Enable the BGP sessions to carry VPN-IPv6 addresses
===================================================

First, we enable the BGP sessions on the route-reflectors for the proper address family:

```
set protocols bgp group rr family inet6-vpn unicast
```

After this, we move to `ios_xr_1`:

```
router bgp 1
 address-family vpnv6 unicast
 !
 neighbor-group rr-client
  address-family vpnv6 unicast
   soft-reconfiguration inbound always
! 
 vrf cust-1
  address-family ipv6 unicast
   redistribute connected
   redistribute static
  !
 !
!
```

Notice the `vrf cust-1` part of the configuration. Same as with the IPv4 addresses previously, we need to specify the IOS XR device to advertise the static and connected routes.

On `vmx5`, we add the following:

```
set protocols bgp group rr-client family inet6-vpn unicast
```

To verify the BGP session with the RR from `ios_xr_1`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show bgp neighbor 10.0.0.15 | include VPNv6</b>
    Address family VPNv6 Unicast: advertised and received
</pre>

To verify the same thig on `vmx5`, we issue the following command:

<pre>
salt@vmx01:r5> <b>show bgp neighbor 10.0.0.15 | match NLRI.*session</b>
  NLRI for this session: inet-vpn-unicast inet6-vpn-unicast
</pre>

















===================================
4: routes are hidden on RR:
===================================


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
bgp.l3vpn-inet6.0    
                       4          0          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.5                  1         37         42       0       1       13:59 Establ
  bgp.l3vpn-inet6.0: 0/1/1/0
  bgp.l3vpn.0: 2/2/2/0
10.0.0.6                  1         24         27       0       1        8:14 Establ
  bgp.l3vpn-inet6.0: 0/1/1/0
  bgp.l3vpn.0: 2/2/2/0
10.0.1.1                  1         13         14       0       1        3:39 Establ
  bgp.l3vpn-inet6.0: 0/1/1/0
  bgp.l3vpn.0: 2/2/2/0
10.0.1.2                  1          6          5       0       1          36 Establ
  bgp.l3vpn-inet6.0: 0/1/1/0
  bgp.l3vpn.0: 2/2/2/0

salt@vmx01:r15> show route table bgp.l3vpn-inet6.0 

bgp.l3vpn-inet6.0: 4 destinations, 4 routes (0 active, 0 holddown, 4 hidden)

salt@vmx01:r15> show route table bgp.l3vpn-inet6.0 hidden 

bgp.l3vpn-inet6.0: 4 destinations, 4 routes (0 active, 0 holddown, 4 hidden)
+ = Active Route, - = Last Active, * = Both

1:1:2001:db8:1::/127               
                    [BGP/170] 00:03:46, MED 0, localpref 100, from 10.0.1.1
                      AS path: ?, validation-state: unverified
                       Unusable
1:1:2001:db8:1::2/127               
                    [BGP/170] 00:00:24, MED 0, localpref 100, from 10.0.1.2
                      AS path: ?, validation-state: unverified
                       Unusable
1:1:2001:db8:1::4/127               
                    [BGP/170] 00:14:11, localpref 100, from 10.0.0.5
                      AS path: I, validation-state: unverified
                       Unusable
1:1:2001:db8:1::6/127               
                    [BGP/170] 00:08:26, localpref 100, from 10.0.0.6
                      AS path: I, validation-state: unverified
                       Unusable
```   

Check the next-hop:

```
salt@vmx01:r15> show route table bgp.l3vpn-inet6.0 hidden extensive 2001:db8:1::/127        

bgp.l3vpn-inet6.0: 4 destinations, 4 routes (0 active, 0 holddown, 4 hidden)
1:1:2001:db8:1::/127 (1 entry, 0 announced)
         BGP    Preference: 170/-101
                Route Distinguisher: 1:1
                Next hop type: Unusable, Next hop index: 0
                Address: 0xbc50f10
                Next-hop reference count: 4
                State: <Hidden Int Ext Changed ProtectionPath ProtectionCand>
                Local AS:     1 Peer AS:     1
                Age: 4:41       Metric: 0 
                Validation State: unverified 
                Task: BGP_1.10.0.1.1
                AS path: ? 
                Communities: target:1:1
                Accepted
                VPN Label: 24010
                Localpref: 100
                Router ID: 10.0.1.1
                Indirect next hops: 1
                        Protocol next hop: ::ffff:10.0.1.1
                        Label operation: Push 24010
                        Label TTL action: prop-ttl
                        Load balance label: Label 24010: None; 
                        Indirect next hop: 0x0 - INH Session ID: 0x0
```         


===================================
5: enable v6 tunneling on Junipers:
===================================


```
salt@vmx01:r15> show route table inet6.3 

salt@vmx01:r15> show route 10.0.1.1         

inet.0: 27 destinations, 27 routes (27 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.1/32        *[OSPF/10] 3d 12:11:45, metric 201
                    >  to 192.168.15.0 via ge-0/0/2.15

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.1.1/32        *[LDP/9] 00:00:48, metric 201
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 181

salt@vmx01:r15> configure 
Entering configuration mode

[edit]
salt@vmx01:r15# set protocols mpls ipv6-tunneling 

[edit]
salt@vmx01:r15# commit and-quit 
commit complete
Exiting configuration mode

salt@vmx01:r15> show route table inet6.3    

inet6.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

::ffff:10.0.0.1/128*[LDP/9] 00:00:02, metric 100
                    >  to 192.168.15.0 via ge-0/0/2.15
::ffff:10.0.0.2/128*[LDP/9] 00:00:02, metric 200
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 173
::ffff:10.0.0.3/128*[LDP/9] 00:00:02, metric 300
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 174
::ffff:10.0.0.4/128*[LDP/9] 00:00:02, metric 200
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 175
::ffff:10.0.0.5/128*[LDP/9] 00:00:02, metric 300
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 176
::ffff:10.0.0.6/128*[LDP/9] 00:00:02, metric 300
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 177
::ffff:10.0.0.14/128
                   *[LDP/9] 00:00:02, metric 300
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 178
::ffff:10.0.1.1/128*[LDP/9] 00:00:02, metric 201
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 181
::ffff:10.0.1.2/128*[LDP/9] 00:00:02, metric 201
                    >  to 192.168.15.0 via ge-0/0/2.15, Push 182
```                    

After activating it on RRs, Cisco is already working:

```
RP/0/RP0/CPU0:ios_xr_1#show route vrf cust-1 ipv6 
Fri Aug 30 07:40:56.091 UTC

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

C    2001:db8:1::/127 is directly connected,
      00:25:09, GigabitEthernet0/0/0/3.2002
L    2001:db8:1::1/128 is directly connected,
      00:25:09, GigabitEthernet0/0/0/3.2002
B    2001:db8:1::2/127 
      [200/0] via ::ffff:10.0.1.2 (nexthop in vrf default), 00:01:09
B    2001:db8:1::4/127 
      [200/0] via ::ffff:10.0.0.5 (nexthop in vrf default), 00:01:09
B    2001:db8:1::6/127 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 00:01:09
```      


Juniper is still hidden on PE:

```
salt@vmx01:r5> show route table cust-1.inet6.0 hidden           

cust-1.inet6.0: 7 destinations, 7 routes (4 active, 0 holddown, 3 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:1::/127    [BGP/170] 00:02:12, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                       Unusable
2001:db8:1::2/127   [BGP/170] 00:02:12, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                       Unusable
2001:db8:1::6/127   [BGP/170] 00:02:12, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                       Unusable
```          

On vmx5 and vmx6:
```
set protocols mpls ipv6-tunneling
```

Then we see the following:

```
salt@vmx01:r5> show route table cust-1.inet6.0           

cust-1.inet6.0: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8:1::/127   *[BGP/170] 00:00:31, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24010, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24010, Push 333(top)
2001:db8:1::2/127  *[BGP/170] 00:00:31, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24010, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24010, Push 334(top)
2001:db8:1::4/127  *[Direct/0] 00:27:47
                    >  via ge-0/0/1.2000
2001:db8:1::5/128  *[Local/0] 00:27:47
                       Local via ge-0/0/1.2000
2001:db8:1::6/127  *[BGP/170] 00:00:31, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
```       

========================================
6: adding statics
====================



router static
 vrf cust-1
  address-family ipv6 unicast
   2001:db8::1/128 2001:db8:1::
  !
 !
!

router static
 vrf cust-1
  address-family ipv6 unicast
   2001:db8::2/128 2001:db8:1::2
  !
 !
!

vmx5:

 set routing-instances cust-1 routing-options rib cust-1.inet6.0 static route 2001:db8::3/128 next-hop 2001:db8:1::4

vmx6:

set routing-instances cust-1 routing-options rib cust-1.inet6.0 static route 2001:db8::4/128 next-hop 2001:db8:1::6


Verifying:

```
salt@vmx01> ping routing-instance c1-1 rapid source 2001:DB8::1 2001:DB8::2    
PING6(56=40+8+8 bytes) 2001:db8::1 --> 2001:db8::2
!!!!!
--- 2001:DB8::2 ping6 statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 7.586/8.835/9.695/0.687 ms

salt@vmx01> ping routing-instance c1-1 rapid source 2001:DB8::1 2001:DB8::3    
PING6(56=40+8+8 bytes) 2001:db8::1 --> 2001:db8::3
!!!!!
--- 2001:DB8::3 ping6 statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 3.749/3.948/4.331/0.209 ms

salt@vmx01> ping routing-instance c1-1 rapid source 2001:DB8::1 2001:DB8::4    
PING6(56=40+8+8 bytes) 2001:db8::1 --> 2001:db8::4
!!!!!
--- 2001:DB8::4 ping6 statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 3.595/3.747/3.807/0.082 ms
```



```
RP/0/RP0/CPU0:ios_xr_1#show route vrf cust-1 ipv6
Fri Aug 30 08:03:16.320 UTC

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

S    2001:db8::1/128 
      [1/0] via 2001:db8:1::, 00:17:36
B    2001:db8::2/128 
      [200/0] via ::ffff:10.0.1.2 (nexthop in vrf default), 00:16:20
B    2001:db8::3/128 
      [200/0] via ::ffff:10.0.0.5 (nexthop in vrf default), 00:02:40
B    2001:db8::4/128 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 00:02:11
C    2001:db8:1::/127 is directly connected,
      00:47:28, GigabitEthernet0/0/0/3.2002
L    2001:db8:1::1/128 is directly connected,
      00:47:28, GigabitEthernet0/0/0/3.2002
B    2001:db8:1::2/127 
      [200/0] via ::ffff:10.0.1.2 (nexthop in vrf default), 00:23:29
B    2001:db8:1::4/127 
      [200/0] via ::ffff:10.0.0.5 (nexthop in vrf default), 00:23:29
B    2001:db8:1::6/127 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 00:23:29
```

```
salt@vmx01:r5> show route table cust-1.inet6.0 

cust-1.inet6.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8::1/128    *[BGP/170] 00:17:11, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24011, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24011, Push 333(top)
2001:db8::2/128    *[BGP/170] 00:15:55, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24011, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24011, Push 334(top)
2001:db8::3/128    *[Static/5] 00:02:16
                    >  to 2001:db8:1::4 via ge-0/0/1.2000
2001:db8::4/128    *[BGP/170] 00:01:46, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
2001:db8:1::/127   *[BGP/170] 00:20:30, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24010, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24010, Push 333(top)
2001:db8:1::2/127  *[BGP/170] 00:20:30, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24010, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24010, Push 334(top)
2001:db8:1::4/127  *[Direct/0] 00:47:46
                    >  via ge-0/0/1.2000
2001:db8:1::5/128  *[Local/0] 00:47:46
                       Local via ge-0/0/1.2000
2001:db8:1::6/127  *[BGP/170] 00:20:30, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
fe80::5254:7:d07c:df65/128
                   *[Local/0] 00:47:46
                       Local via ge-0/0/1.2000
ff02::2/128        *[INET6/0] 1w2d 01:24:30
                       MultiRecv
```                       











Cisco:

show bgp vpnv6 unicast neighbors 10.0.0.15 received routes
show bgp vpnv6 unicast neighbors 10.0.0.15 advertised-routes 
show bgp vpnv6 unicast vrf cust-1 2001:db8:1::4/127 detail 
show route vrf cust-1 ipv6
