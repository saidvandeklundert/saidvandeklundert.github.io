---
layout: post
title: MPLS L3VPN on Cisco IOSXR and Juniper MX using BGP
tags: [juniper, cisco, mpls, l3vpn]
image: /img/cisco_juniper_logo.png
---


Using the network I created [previously](www.saidvandeklundert.net/2019-08-28-juniper-cisco_ios_xr_mpls_vpn/), in this post, I am going to create a basic MPLS L3VPN between a Cisco IOSXR PE and a Juniper MX PE and use BGP between the CPE and the PE:


![MPLS L3VPN with BGP](/img/cisco_iosxr_mpls_l3pnv_bgp.png "MPLS L3VPN with BGP")





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


