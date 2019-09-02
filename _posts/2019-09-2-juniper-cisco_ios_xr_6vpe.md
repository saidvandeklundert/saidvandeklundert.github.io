---
layout: post
title: 6VPE between Juniper MX and Cisco IOS XR
image: /img/cisco_juniper_logo.png
---

In this post, we will enable the a VPN for IPv6 using 6VPE. We will use the same topology used during my last post: [MPLS L3VPN between Juniper MX and Cisco IOS XR](http://saidvandeklundert.net/2019-08-28-juniper-cisco_ios_xr_mpls_vpn/ ), where we used the following topology: 

![juniper and ios xr topology](/img/topology_juniper_ios_xr.png "topology")

For 6VPE, we will create the following VPN topology:

![juniper and ios xr 6vpe vpn topology](/img/vpn_topology_juniper_ios_xr_6vpe.png "6vpe vpn topology")


The steps to get to this working VPN are as follows:
- enable the CE-facing PE interfaces for IPv6
- configure the IPv6 static routes
- ready the VRF configuration for IPv6
- enable the BGP sessions to carry VPN-IPv6 addresses
- enable IPv6 tunneling
- verify the 6VPE VPN


<br>

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

<br>

Configuring the static routes
=============================

To configure a static IPv6 route in the `cust-1` VRF on `ios_xr_1`, we apply the following configuration:

```
router static
 vrf cust-1
  address-family ipv6 unicast
   2001:db8::1/128 2001:db8:1::
  !
 !
```

On `vmx5`, we can do the same thing using the following:

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

<br>

Ready the VRF configuration for IPv6
====================================


On `ios_xr_1`, we add the following configuration to the vrf to allow the import and export of IPv6 routes:

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

The `vmx` requires no such configuration. After enabling the interface in the VRF for IPv6, we can see the following:

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


<br>

Enable the BGP sessions to carry VPN-IPv6 addresses
===================================================

The BGP sessions between the PEs and the RRs have been configured to carry MPLS VPN routes previously. We now need to enable the BGP sessions for an additional address family so that the PEs can carry VPN-IPv6 routes. We need to add the address family to the following BGP sessions:

![juniper and ios xr BGP configuration](/img/rr_bgp_configuration.png "BGP configuration")

First, we enable the BGP sessions on the route-reflectors for the proper address family. On `vmx14` and `vmx15`, we add the following configuration:

```
set protocols bgp group rr family inet6-vpn unicast
```

After this, we move to `ios_xr_1` and `ios_xr_2` where we add the following:

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

On `vmx5` and `vmx6`, we add the following configuration:

```
set protocols bgp group rr-client family inet6-vpn unicast
```

To verify the BGP session with the RR from `ios_xr_1`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show bgp neighbor 10.0.0.15 | include VPNv6</b>
    Address family VPNv6 Unicast: advertised and received
</pre>

To verify the same thing on `vmx5`, we issue the following command:

<pre>
salt@vmx01:r5> <b>show bgp neighbor 10.0.0.15 | match NLRI.*session</b>
  NLRI for this session: inet-vpn-unicast inet6-vpn-unicast
</pre>


<br>

Enable IPv6 tunneling
=====================

When the PEs advertise the IPv6 route for the VRF, the next-hop attribute is an IPv4-mapped IPv6 address that is automatically derived from the loopback IP address of the advertising PE. 

When we configure IPv6 tunneling on a Juniper device, we copy all the IPv4 destinations from the `inet.3` table to the `inet6.3` table. This step is required on Juniper devices only. No equivalent configuration is required to enable an IOS XR to resolve IPv4 mapped IPv6 addresses.

The only device participating with 6VPE are the RRs and the PEs. For that reasons, on `vmx5`, `vmx6`, `vmx14` and `vmx15`, we configure the following:

```
set protocols mpls ipv6-tunneling
```

After enabling IPv6 tunneling, we can use the following commands to verify that the IPv4 addresses have been copied over to the `inet6.3` table on `vmx5`:

<pre>
salt@vmx5> <b>show route table inet.3</b>

inet.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.1/32        *[LDP/9] 3d 06:14:59, metric 200
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 321
10.0.0.2/32        *[LDP/9] 3d 06:14:59, metric 200
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 324
10.0.0.3/32        *[LDP/9] 3d 06:14:59, metric 100
                    >  to 192.168.7.1 via ge-0/0/1.7
10.0.0.4/32        *[LDP/9] 3d 06:14:59, metric 100
                    >  to 192.168.5.1 via ge-0/0/1.5
10.0.0.6/32        *[LDP/9] 3d 06:14:59, metric 200
                       to 192.168.5.1 via ge-0/0/1.5, Push 319
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 327
10.0.0.14/32       *[LDP/9] 3d 06:14:59, metric 200
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 320
10.0.0.15/32       *[LDP/9] 3d 06:14:59, metric 300
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 322
10.0.1.1/32        *[LDP/9] 3d 06:14:59, metric 301
                       to 192.168.5.1 via ge-0/0/1.5, Push 325
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 333
10.0.1.2/32        *[LDP/9] 3d 06:14:59, metric 301
                       to 192.168.5.1 via ge-0/0/1.5, Push 326
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 334

salt@vmx5> <b>show route table inet6.3</b>

inet6.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

::ffff:10.0.0.1/128*[LDP/9] 3d 06:15:02, metric 200
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 321
::ffff:10.0.0.2/128*[LDP/9] 3d 06:15:02, metric 200
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 324
::ffff:10.0.0.3/128*[LDP/9] 3d 06:15:02, metric 100
                    >  to 192.168.7.1 via ge-0/0/1.7
::ffff:10.0.0.4/128*[LDP/9] 3d 06:15:02, metric 100
                    >  to 192.168.5.1 via ge-0/0/1.5
::ffff:10.0.0.6/128*[LDP/9] 3d 06:15:02, metric 200
                       to 192.168.5.1 via ge-0/0/1.5, Push 319
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 327
::ffff:10.0.0.14/128
                   *[LDP/9] 3d 06:15:02, metric 200
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 320
::ffff:10.0.0.15/128
                   *[LDP/9] 3d 06:15:02, metric 300
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 322
::ffff:10.0.1.1/128*[LDP/9] 3d 06:15:02, metric 301
                       to 192.168.5.1 via ge-0/0/1.5, Push 325
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 333
::ffff:10.0.1.2/128*[LDP/9] 3d 06:15:02, metric 301
                       to 192.168.5.1 via ge-0/0/1.5, Push 326
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 334
</pre>


<br>

Verify the 6VPE VPN
===================

To verify 6VPE operations, we focus on `ios_xr_1` and `vmx5` and start off checking the routes that have been exchanged with the `vmx15` route-reflector.

![juniper and ios xr Verify the 6VPE VPN](/img/vpn_topology_juniper_ios_xr_6vpe_ios_xr_1_vmx_5.png "Verify the 6VPE VPN")

On `ios_xr_1`, we check the following:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show bgp vpnv6 unicast neighbors 10.0.0.15 advertised-routes</b>
Mon Sep  2 14:08:36.877 UTC
Network            Next Hop        From            AS Path
Route Distinguisher: 1:1
2001:db8::1/128    10.0.1.1        Local           ?
2001:db8:1::/127   10.0.1.1        Local           ?

Processed 2 prefixes, 2 paths

RP/0/RP0/CPU0:ios_xr_1#<b>show bgp vpnv6 unicast neighbors 10.0.0.15 received routes</b>
Mon Sep  2 14:08:39.313 UTC
BGP router identifier 10.0.1.1, local AS number 1
BGP generic scan interval 60 secs
Non-stop routing is enabled
BGP table state: Active
Table ID: 0x0   RD version: 0
BGP main routing table version 74
BGP NSR Initial initsync version 4 (Reached)
BGP NSR/ISSU Sync-Group versions 0/0
BGP scan interval 60 secs

Status codes: s suppressed, d damped, h history, * valid, > best
              i - internal, r RIB-failure, S stale, N Nexthop-discard
Origin codes: i - IGP, e - EGP, ? - incomplete
   Network            Next Hop            Metric LocPrf Weight Path
Route Distinguisher: 1:1 (default for vrf cust-1)
*>i2001:db8::2/128    10.0.1.2                 0    100      0 ?
*>i2001:db8::3/128    10.0.0.5                      100      0 i
*>i2001:db8::4/128    10.0.0.6                      100      0 i
*>i2001:db8:1::2/127  10.0.1.2                 0    100      0 ?
*>i2001:db8:1::4/127  10.0.0.5                      100      0 i
*>i2001:db8:1::6/127  10.0.0.6                      100      0 i

Processed 6 prefixes, 6 paths
</pre>

We see the `ios_xr_1` device has advertised hte connected as well as the static route. In additional to that, we see all the other routes in the VPN have been learned from the route reflector.

To verify the same thing on `vmx5`, we issue the following command:

<pre>
salt@vmx05> <b>show route advertising-protocol bgp 10.0.0.15 table cust-1.inet6.0</b>

cust-1.inet6.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 2001:db8::3/128         Self                         100        I
* 2001:db8:1::4/127       Self                         100        I

salt@vmx5> <b>show route receive-protocol bgp 10.0.0.15 table cust-1.inet6.0</b>

cust-1.inet6.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 2001:db8::1/128         ::ffff:10.0.1.1      0       100        ?
* 2001:db8::2/128         ::ffff:10.0.1.2      0       100        ?
* 2001:db8::4/128         ::ffff:10.0.0.6              100        I
* 2001:db8:1::/127        ::ffff:10.0.1.1      0       100        ?
* 2001:db8:1::2/127       ::ffff:10.0.1.2      0       100        ?
* 2001:db8:1::6/127       ::ffff:10.0.0.6              100        I
</pre>


To check the routing table on for the routes on `ios_xr_1`, we issue the following command:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show route vrf cust-1 ipv6</b>

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
      [1/0] via 2001:db8:1::, 3d06h
B    2001:db8::2/128 
      [200/0] via ::ffff:10.0.1.2 (nexthop in vrf default), 2d03h
B    2001:db8::3/128 
      [200/0] via ::ffff:10.0.0.5 (nexthop in vrf default), 2d03h
B    2001:db8::4/128 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 2d03h
C    2001:db8:1::/127 is directly connected,
      3d06h, GigabitEthernet0/0/0/3.2002
L    2001:db8:1::1/128 is directly connected,
      3d06h, GigabitEthernet0/0/0/3.2002
B    2001:db8:1::2/127 
      [200/0] via ::ffff:10.0.1.2 (nexthop in vrf default), 2d03h
B    2001:db8:1::4/127 
      [200/0] via ::ffff:10.0.0.5 (nexthop in vrf default), 2d03h
B    2001:db8:1::6/127 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 2d03h
</pre>

Verifying the routing table on `vmx5` is done like so:

<pre>
salt@vmx5> <b>show route table cust-1.inet6.0</b>

cust-1.inet6.0: 11 destinations, 11 routes (11 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8::1/128    *[BGP/170] 2d 04:05:35, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24011, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24011, Push 333(top)
2001:db8::2/128    *[BGP/170] 2d 04:05:33, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24011, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24011, Push 334(top)
2001:db8::3/128    *[Static/5] 3d 06:06:39
                    >  to 2001:db8:1::4 via ge-0/0/1.2000
2001:db8::4/128    *[BGP/170] 2d 04:05:31, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
2001:db8:1::/127   *[BGP/170] 2d 04:05:35, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24010, Push 325(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24010, Push 333(top)
2001:db8:1::2/127  *[BGP/170] 2d 04:05:33, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 24010, Push 326(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 24010, Push 334(top)
2001:db8:1::4/127  *[Direct/0] 3d 06:52:09
                    >  via ge-0/0/1.2000
2001:db8:1::5/128  *[Local/0] 3d 06:52:09
                       Local via ge-0/0/1.2000
2001:db8:1::6/127  *[BGP/170] 2d 04:05:31, localpref 100, from 10.0.0.15
                      AS path: I, validation-state: unverified
                    >  to 192.168.5.1 via ge-0/0/1.5, Push 273, Push 319(top)
                       to 192.168.7.1 via ge-0/0/1.7, Push 273, Push 327(top)
fe80::5254:7:d07c:df65/128
                   *[Local/0] 3d 06:52:09
                       Local via ge-0/0/1.2000
ff02::2/128        *[INET6/0] 1w5d 07:28:53
                       MultiRecv
</pre>

Both `ios_xr_1` as well as `vmx5` advertised a static route to the RRs. Before checking the forwarding entries on `ios_xr_1` and `vmx5`, we check the routes in deailt on the `vmx15` RR:

<pre>
salt@vmx15> <b>show route receive-protocol bgp 10.0.1.1 2001:db8::1/128 detail</b>

bgp.l3vpn-inet6.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
* 1:1:<b>2001:db8::1</b> /128 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 1:1
     VPN Label: <b>24011</b> 
     Nexthop: <b>::ffff:10.0.1.1</b> 
     MED: 0
     Localpref: 100
     AS path: ? 
     Communities: target:1:1

salt@vmx15> <b>show route receive-protocol bgp 10.0.0.5 2001:db8::3/128 detail</b> 

bgp.l3vpn-inet6.0: 8 destinations, 8 routes (8 active, 0 holddown, 0 hidden)
* 1:1:<b>2001:db8::3</b> /128 (1 entry, 1 announced)
     Accepted
     Route Distinguisher: 1:1
     VPN Label: <b>282</b> 
     Nexthop: <b>::ffff:10.0.0.5</b> 
     Localpref: 100
     AS path: I 
     Communities: target:1:1
</pre>

To check an entry in the forwarding table on `ios_xr_1`:

<pre>
RP/0/RP0/CPU0:ios_xr_1#<b>show cef vrf cust-1 ipv6  2001:db8::3/128</b>

2001:db8::3/128, version 39, internal 0x5000001 0x0 (ptr 0xe369808) [1], 0x0 (0xe529968), 0xa08 (0xe7106a8)
 Updated Aug 31 10:01:43.825
 Prefix Len 128, traffic index 0, precedence n/a, priority 3
   via ::ffff:10.0.0.5/128, 5 dependencies, recursive [flags 0x6000]
    path-idx 0 NHID 0x0 [0xd436d90 0x0]
    recursion-via-/128
    next hop VRF - 'default', table - 0xe0000000
    next hop ::ffff:10.0.0.5/128 via 24004/0/21
     next hop 10.0.2.0/32 Gi0/0/0/1.10 labels imposed {176 282}
     next hop 10.0.2.4/32 Gi0/0/0/2.12 labels imposed {340 282}

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

From the RR output, we saw that the VPN label to reach the prefix on `C1-3` was `282`. From the CEF output, we can see that this label is imposed as the VPN label. To learn where the transport labels came from, we checked the LDP route towards `vmx5` where we saw both inner labels.

To check the same thing on `vmx5`:

<pre>
salt@vmx5> <b>show route forwarding-table vpn cust-1 destination 2001:db8::1/128</b>
Routing table: cust-1.inet6
Internet6:
Enabled protocols: Bridging, All VLANs, 
Destination        Type RtRef Next hop           Type Index    NhRef Netif
2001:db8::1/128    user     0                    indr  1048610     2
                                                 ulst  1048577     2
                              192.168.5.1       Push 24011, Push 325(top)     1543     2 ge-0/0/1.5
                              192.168.7.1       Push 24011, Push 333(top)     1692     2 ge-0/0/1.7

salt@vmx01:r5> <b>show route table inet6.3 ::ffff:10.0.1.1/128</b>    

inet6.3: 9 destinations, 9 routes (9 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

::ffff:10.0.1.1/128*[LDP/9] 3d 06:55:11, metric 301
                       to 192.168.5.1 via ge-0/0/1.5, Push 325
                    >  to 192.168.7.1 via ge-0/0/1.7, Push 333                              
</pre>                              

From the RR output earlier, we learned that `ios_xr_1` was advertising `2001:db8::1/128` with VPN label `24011`. The `show route forwarding-table` command reveaveled that the `vmx` has installed this label in addition to the LDP learned labels towards `ios_xr_1`. 


With all the routing and forwarding entries in place, we finish up verifying that we can exchange ICMPs between the different CPEs:

<pre>
salt@vmx01> <b>ping routing-instance c1-1 rapid source 2001:db8::1 2001:db8::2</b>      
PING6(56=40+8+8 bytes) 2001:db8::1 --> 2001:db8::2
!!!!!
--- 2001:db8::2 ping6 statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 1.952/2.091/2.217/0.091 ms

salt@vmx01> <b>ping routing-instance c1-1 rapid source 2001:db8::1 2001:db8::3</b>    
PING6(56=40+8+8 bytes) 2001:db8::1 --> 2001:db8::3
!!!!!
--- 2001:db8::3 ping6 statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 3.424/3.555/3.737/0.129 ms

salt@vmx01> <b>ping routing-instance c1-1 rapid source 2001:db8::1 2001:db8::4</b>    
PING6(56=40+8+8 bytes) 2001:db8::1 --> 2001:db8::4
!!!!!
--- 2001:db8::4 ping6 statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/std-dev = 3.546/14.598/25.601/8.149 ms
</pre>


Review the complete VPN configuration
=====================================





After