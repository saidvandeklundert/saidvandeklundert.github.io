---
layout: post
title: MPLS L3VPN on Cisco IOS XR and Juniper MX using BGP
tags: [juniper, cisco, mpls, l3vpn]
image: /img/cisco_juniper_logo.png
---


Using the network I created [previously](www.saidvandeklundert.net/2019-08-28-juniper-cisco_ios_xr_mpls_vpn/), in this post, I am going to create a basic MPLS L3VPN between a <b>Cisco IOS XR</b> and a <b>Juniper MX</b> and use BGP between the CPE and the PE:


![MPLS L3VPN with BGP](/img/cisco_iosxr_mpls_l3pnv_bgp.png "MPLS L3VPN with BGP")

At first, I will share the complete configuration. I will use <b>show configuration | display set</b> on the Juniper MX device and <b>show running-config formal</b> on the Cisco IOS XR device.

After this, I will do a walkthrough of the configuration and I will finish up verifying everything.



The configuration:
==================

<b>ios_xr_1</b>:

<pre style="font-size:12px">
interface GigabitEthernet0/0/0/2.2004 description c2-1
interface GigabitEthernet0/0/0/2.2004 vrf cust-2
interface GigabitEthernet0/0/0/2.2004 ipv4 address 10.0.0.17 255.255.255.252
interface GigabitEthernet0/0/0/2.2004 ipv6 address 2001:db8:1::9/127
interface GigabitEthernet0/0/0/2.2004 encapsulation dot1q 2004

vrf cust-2 rd 2:2
vrf cust-2 address-family ipv4 unicast import route-target 2:2
vrf cust-2 address-family ipv4 unicast export route-target 2:2
vrf cust-2 address-family ipv6 unicast import route-target 2:2
vrf cust-2 address-family ipv6 unicast export route-target 2:2

route-policy ACCEPT
  pass
end-policy


router bgp 1 vrf cust-2 address-family ipv4 unicast redistribute connected
router bgp 1 vrf cust-2 address-family ipv6 unicast redistribute connected

router bgp 1 vrf cust-2 neighbor 10.0.0.18 remote-as 65000
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast route-policy ACCEPT in
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast route-policy ACCEPT out
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast as-override
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast soft-reconfiguration inbound always

router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 remote-as 65000
router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 address-family ipv6 unicast route-policy ACCEPT in
router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 address-family ipv6 unicast route-policy ACCEPT out
router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 address-family ipv6 unicast as-override
</pre>

<b>vmx6</b>:

<pre style="font-size:12px">
set interfaces ge-0/0/1 unit 2007 description c2-4
set interfaces ge-0/0/1 unit 2007 vlan-id 2007
set interfaces ge-0/0/1 unit 2007 family inet address 10.0.0.29/30
set interfaces ge-0/0/1 unit 2007 family inet6 address 2001:db8:1::15/127

set routing-instances cust-2 instance-type vrf
set routing-instances cust-2 interface ge-0/0/1.2007
set routing-instances cust-2 route-distinguisher 2:2
set routing-instances cust-2 vrf-target target:2:2
set routing-instances cust-2 vrf-table-label

set routing-instances cust-2 protocols bgp group cpe-v4 peer-as 65000
set routing-instances cust-2 protocols bgp group cpe-v4 as-override
set routing-instances cust-2 protocols bgp group cpe-v4 neighbor 10.0.0.30

set routing-instances cust-2 protocols bgp group cpe-v6 peer-as 65000
set routing-instances cust-2 protocols bgp group cpe-v6 as-override
set routing-instances cust-2 protocols bgp group cpe-v6 neighbor 2001:db8:1::14
</pre>


Configuration walkthrough:
==========================


<b>Cisco</b>:

We create the VRF and specify the route-distinguisher. Additionally, we specify the import and export target for both the IPv4 as well as the IPv6 address family: 

<pre style="font-size:12px">
vrf cust-2 rd 2:2
vrf cust-2 address-family ipv4 unicast import route-target 2:2
vrf cust-2 address-family ipv4 unicast export route-target 2:2
vrf cust-2 address-family ipv6 unicast import route-target 2:2
vrf cust-2 address-family ipv6 unicast export route-target 2:2
</pre>

Instead of specifying a route-target like this, it would have also been possible to use a route-policy to determine what the route-target should be. This is something worth considering if you are creating a more complex VPN topology that actually requires that, like a hub-and-spoke VPN for instance. In this case though, we are keeping it simple. 

After having created the VRF, we configure the interface and assign it to the VRF:

<pre style="font-size:12px">
interface GigabitEthernet0/0/0/2.2004 description c2-1
interface GigabitEthernet0/0/0/2.2004 vrf cust-2
interface GigabitEthernet0/0/0/2.2004 ipv4 address 10.0.0.17 255.255.255.252
interface GigabitEthernet0/0/0/2.2004 ipv6 address 2001:db8:1::9/127
interface GigabitEthernet0/0/0/2.2004 encapsulation dot1q 2004
</pre>

We must have a route-policy in place for the IOS XR BGP sessions to accept any route import or export. To this end, we create the following route-policy before we move on to the BGP section:

<pre style="font-size:12px">
route-policy ACCEPT
  pass
end-policy
</pre>

Next is the actual BGP session and the options. All the customer BGP configuration for the VRF happens in the <b>router bgp 1 vrf cust-2</b> configuration stanza. We want the PE to advertise the PE-CPE link to other PEs for both address families, so we instruct the device to advertise the connected subnets:

<pre style="font-size:12px">
router bgp 1 vrf cust-2 address-family ipv4 unicast redistribute connected
router bgp 1 vrf cust-2 address-family ipv6 unicast redistribute connected
</pre>

After this, we configure the IPv4 BGP session. We specify the remote-as and the route-policy. Additionally, we enable soft-reconfiguration inbound. This way, we can inspect the received routes from the neighbor, a behavior that is on by default on a Juniper device. To do this for both IPv4 as well as IPv6, we configure the following:

<pre style="font-size:12px">
router bgp 1 vrf cust-2 neighbor 10.0.0.18 remote-as 65000
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast route-policy ACCEPT in
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast route-policy ACCEPT out
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast as-override
router bgp 1 vrf cust-2 neighbor 10.0.0.18 address-family ipv4 unicast soft-reconfiguration inbound always

router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 remote-as 65000
router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 address-family ipv6 unicast route-policy ACCEPT in
router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 address-family ipv6 unicast route-policy ACCEPT out
router bgp 1 vrf cust-2 neighbor 2001:db8:1::8 address-family ipv6 unicast as-override
</pre>

<b>Juniper</b>:

We configure the interface and the VPN, together with the route-target and route-distinguisher:

<pre style="font-size:12px">
set interfaces ge-0/0/1 unit 2007 description c2-4
set interfaces ge-0/0/1 unit 2007 vlan-id 2007
set interfaces ge-0/0/1 unit 2007 family inet address 10.0.0.29/30
set interfaces ge-0/0/1 unit 2007 family inet6 address 2001:db8:1::15/127

set routing-instances cust-2 instance-type vrf
set routing-instances cust-2 interface ge-0/0/1.2007
set routing-instances cust-2 route-distinguisher 2:2
set routing-instances cust-2 vrf-target target:2:2
set routing-instances cust-2 vrf-table-label
</pre>

It is possible to explicitly configure an import and an export policy in Juniper as well. In our case though, to keep it simple, we used <b>vrf-target</b>. This will ensure the referenced route-target is attached to IPv4 as well as IPv6 routes. Additionally, it will make sure that routes tagged with that target are imported into the vrf. After configuring <b>vrf-target</b>, Juniper will actually create several <i>hidden</i> route policies that allows you to see exactly what is going on:

<pre style="font-size:12px">
salt@vmx6> <b>show policy ?</b>  
Configured policies:
__vrf-export-cust-1-internal__
<b>__vrf-export-cust-2-internal__</b>
__vrf-import-cust-1-internal__
<b>__vrf-import-cust-2-internal__</b>
lbpp

salt@vmx6> <b>show policy __vrf-export-cust-2-internal__</b> 
Policy __vrf-export-cust-2-internal__:
    Term unnamed:
        <b>then community + __vrf-community-cust-2-common-internal__ [target:2:2 ] accept</b>

salt@vmx6> <b>show policy __vrf-import-cust-2-internal__</b>
Policy __vrf-import-cust-2-internal__:
    Term unnamed:
        <b>from community __vrf-community-cust-2-common-internal__ [target:2:2 ]</b>
        <b>then accept</b>
    Term unnamed:
        then reject

salt@vmx6>
</pre>

The policies show that routes are tagged with <b>target:2:2</b> on export. Additionally, all received routes with that same extended community will be placed into the <b>cust-2</b> VPN. 

Another thing to note here is that BGP would normally not advertise the connected routes. But since we have <b>vrf-target</b> and <b>vrf-table-label</b> configured under the instance stanza, the PE will be advertising the connected route to the other PE devices.

Next, we configure the BGP sessions for IPv4 and IPv6 inside the routing-instance:

<pre style="font-size:12px">
set routing-instances cust-2 protocols bgp group cpe-v4 peer-as 65000
set routing-instances cust-2 protocols bgp group cpe-v4 as-override
set routing-instances cust-2 protocols bgp group cpe-v4 neighbor 10.0.0.30

set routing-instances cust-2 protocols bgp group cpe-v6 peer-as 65000
set routing-instances cust-2 protocols bgp group cpe-v6 as-override
set routing-instances cust-2 protocols bgp group cpe-v6 neighbor 2001:db8:1::14
</pre>

On Juniper, EBGP learned routes are accepted automatically so we do not need any policy.




Verification:
=============


We start out verifying that the BGP sessions are up on the Cisco device:

<pre style="font-size:12px">
ios_xr_1#<b>show bgp vrf cust-2 summary</b>            
BGP VRF cust-2, state: Active
BGP Route Distinguisher: 2:2

< output omitted >

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
10.0.0.18         0 65000    3127    2874       60    0    0 12:08:10          1

ios_xr_1#<b>show bgp vrf cust-2 ipv6 unicast summary</b>
BGP VRF cust-2, state: Active
BGP Route Distinguisher: 2:2

< output omitted >

Neighbor        Spk    AS MsgRcvd MsgSent   TblVer  InQ OutQ  Up/Down  St/PfxRcd
2001:db8:1::8     0 65000    3020    2776       51    0    0 23:03:07          1
</pre>

To check the same thing on the MX, we issue the following command:

<pre style="font-size:12px">
salt@vmx6> <b>show bgp summary instance cust-2</b>  

< output omitted >

Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State
10.0.0.30             65000       3151       3180       0       0  1d 0:02:28 Establ
  cust-2.inet.0: 1/1/1/0
2001:db8:1::14        65000       3151       3179       0       0  1d 0:02:24 Establ
  cust-2.inet6.0: 1/1/1/0
</pre>


Next, we verify the routing table for the VRF on the Cisco IOS XR device:

<pre style="font-size:12px">
ios_xr_1#<b>show route vrf cust-2</b>

C    10.0.0.16/30 is directly connected, 22:56:35, GigabitEthernet0/0/0/2.2004
L    10.0.0.17/32 is directly connected, 22:56:35, GigabitEthernet0/0/0/2.2004
B    10.0.0.28/30 [200/0] via 10.0.0.6 (nexthop in vrf default), 23:43:08
B    192.168.2.1/32 [20/0] via 10.0.0.18, 11:52:33
B    192.168.2.4/32 [200/0] via 10.0.0.6 (nexthop in vrf default), 23:43:08

ios_xr_1#<b>show route vrf cust-2 ipv6</b>

B    2001:db8::21/128 
      [20/0] via fe80::5254:2:2:2004, 00:01:47, GigabitEthernet0/0/0/2.2004
B    2001:db8::24/128 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 00:01:47
C    2001:db8:1::8/127 is directly connected,
      00:02:20, GigabitEthernet0/0/0/2.2004
L    2001:db8:1::9/128 is directly connected,
      00:02:20, GigabitEthernet0/0/0/2.2004
B    2001:db8:1::14/127 
      [200/0] via ::ffff:10.0.0.6 (nexthop in vrf default), 00:01:47
</pre>

And on the Juniper MX device:

<pre style="font-size:12px">
salt@vmx6> <b>show route table cust-2</b>  

<b>cust-2.inet.0</b>: 5 destinations, 7 routes (5 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.16/30       *[BGP/170] 00:53:51, MED 0, localpref 100, from 10.0.0.14
                      AS path: ?, validation-state: unverified
                       to 192.168.6.1 via ge-0/0/1.6, Push 24015, Push 328(top)
                    >  to 192.168.8.1 via ge-0/0/1.8, Push 24015, Push 336(top)
                    [BGP/170] 00:53:51, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                       to 192.168.6.1 via ge-0/0/1.6, Push 24015, Push 328(top)
                    >  to 192.168.8.1 via ge-0/0/1.8, Push 24015, Push 336(top)
10.0.0.28/30       *[Direct/0] 1d 00:16:05
                    >  via ge-0/0/1.2007
10.0.0.29/32       *[Local/0] 1d 00:16:05
                       Local via ge-0/0/1.2007
192.168.2.1/32     *[BGP/170] 11:53:55, localpref 100, from 10.0.0.14
                      AS path: 65000 I, validation-state: unverified
                       to 192.168.6.1 via ge-0/0/1.6, Push 24012, Push 328(top)
                    >  to 192.168.8.1 via ge-0/0/1.8, Push 24012, Push 336(top)
                    [BGP/170] 11:53:55, localpref 100, from 10.0.0.15
                      AS path: 65000 I, validation-state: unverified
                       to 192.168.6.1 via ge-0/0/1.6, Push 24012, Push 328(top)
                    >  to 192.168.8.1 via ge-0/0/1.8, Push 24012, Push 336(top)
192.168.2.4/32     *[BGP/170] 23:58:47, localpref 100
                      AS path: 65000 I, validation-state: unverified
                    >  to 10.0.0.30 via ge-0/0/1.2007

<b>cust-2.inet6.0</b>: 7 destinations, 9 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2001:db8::21/128   *[BGP/170] 00:03:21, localpref 100, from 10.0.0.14
                      AS path: 65000 I, validation-state: unverified
                    >  to 192.168.6.1 via ge-0/0/1.6, Push 24011, Push 329(top)
                       to 192.168.8.1 via ge-0/0/1.8, Push 24011, Push 337(top)
                    [BGP/170] 00:03:21, localpref 100, from 10.0.0.15
                      AS path: 65000 I, validation-state: unverified
                    >  to 192.168.6.1 via ge-0/0/1.6, Push 24011, Push 329(top)
                       to 192.168.8.1 via ge-0/0/1.8, Push 24011, Push 337(top)
2001:db8::24/128   *[BGP/170] 1w6d 17:10:17, localpref 100
                      AS path: 65000 I, validation-state: unverified
                    >  to 2001:db8:1::14 via ge-0/0/1.2007
2001:db8:1::8/127  *[BGP/170] 00:03:21, MED 0, localpref 100, from 10.0.0.14
                      AS path: ?, validation-state: unverified
                       to 192.168.6.1 via ge-0/0/1.6, Push 24012, Push 329(top)
                    >  to 192.168.8.1 via ge-0/0/1.8, Push 24012, Push 337(top)
                    [BGP/170] 00:03:21, MED 0, localpref 100, from 10.0.0.15
                      AS path: ?, validation-state: unverified
                       to 192.168.6.1 via ge-0/0/1.6, Push 24012, Push 329(top)
                    >  to 192.168.8.1 via ge-0/0/1.8, Push 24012, Push 337(top)
2001:db8:1::14/127 *[Direct/0] 2w0d 16:09:12
                    >  via ge-0/0/1.2007
2001:db8:1::15/128 *[Local/0] 2w0d 16:09:12
                       Local via ge-0/0/1.2007              
</pre>

Notice that this one command shows you both the IPv4 (<b>inet.0</b>) as well as the IPv6 (<b>inet6.0</b>) routing table for this VPN. In case you only want to look at the IPv4 or IPv6 table, you need to issue the <b>show route table cust-2.inet.0</b> or the <b>show route table cust-2.inet6.0</b> command.

As far as the basic checks go, the other thing worth looking in to are the received and advertised routes. To check these things on the Cisco device, we issue the following commands:

<pre style="font-size:12px">
ios_xr_1#<b>show bgp vrf cust-2 neighbors 10.0.0.18 advertised-routes</b>
Network            Next Hop        From            AS Path
Route Distinguisher: 2:2 (default for vrf cust-2)
10.0.0.16/30       10.0.0.17       Local           1?
10.0.0.28/30       10.0.0.17       10.0.0.14       1i
192.168.2.4/32     10.0.0.17       10.0.0.14       1 65000i

Processed 3 prefixes, 3 paths

ios_xr_1#<b>show bgp vrf cust-2 neighbors 10.0.0.18 received routes</b>
< output omitted >
Route Distinguisher: 2:2 (default for vrf cust-2)
*  192.168.2.1/32     10.0.0.18                              0 65000 i

Processed 1 prefixes, 1 paths

ios_xr_1#<b>show bgp vrf cust-2 ipv6 unicast neighbors 2001:db8:1::8 advertised-routes</b>
Network            Next Hop        From            AS Path
Route Distinguisher: 2:2 (default for vrf cust-2)
2001:db8::24/128   2001:db8:1::9   10.0.0.14       1 65000i
2001:db8:1::8/127  2001:db8:1::9   Local           1?
2001:db8:1::14/127 2001:db8:1::9   10.0.0.14       1i

Processed 3 prefixes, 3 paths

 
ios_xr_1#<b>show bgp vrf cust-2 ipv6 unicast neighbors 2001:db8:1::8 received routes</b>  
< output omitted >
Route Distinguisher: 2:2 (default for vrf cust-2)
*  2001:db8::21/128   2001:db8:1::8                          0 65000 i

Processed 1 prefixes, 1 paths
</pre>

To check the same thing on the Juniper device, we use the following:

<pre style="font-size:12px">
salt@vmx6> <b>show route table cust-2 receive-protocol bgp 10.0.0.30</b>   

cust-2.inet.0: 5 destinations, 7 routes (5 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 192.168.2.4/32          10.0.0.30                               65000 I

cust-2.inet6.0: 7 destinations, 9 routes (7 active, 0 holddown, 0 hidden)


salt@vmx6> <b>show route table cust-2 advertising-protocol bgp 10.0.0.30</b>

cust-2.inet.0: 5 destinations, 7 routes (5 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 10.0.0.16/30            Self                                    ?
* 192.168.2.1/32          Self                                    


salt@vmx6> <b>show route table cust-2 receive-protocol bgp 2001:db8:1::14</b>    

cust-2.inet.0: 5 destinations, 7 routes (5 active, 0 holddown, 0 hidden)

cust-2.inet6.0: 7 destinations, 9 routes (7 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 2001:db8::24/128        2001:db8:1::14                          65000 I

salt@vmx6> <b>show route table cust-2 advertising-protocol bgp 2001:db8:1::14</b>

cust-2.inet6.0: 7 destinations, 9 routes (7 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 2001:db8::21/128        Self                                    1 I
* 2001:db8:1::8/127       Self                                    ?
</pre>
 
