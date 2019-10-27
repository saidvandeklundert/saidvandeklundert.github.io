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


Configuration walkthrough:
==========================

Whenever I glance over Juniper configuration, I have always been a fan of the ‘display set’ knob. I think it greatly improves the readability. I was happy to find out recently that in IOSXR, there is something similar. When you use ‘show running-config formal’, you retrieve the configuration of the device in a similar fashion. So using the ‘show configuration | display set’ on the Juniper and using the ‘show running-config formal’ on IOSXR, here goes the walkthrough of the configuration.

<b>Cisco</b>:

We create the VRF and specify the route-distinguisher. Additionally, we specify the import and export target for both the IPv4 as well as the IPv6 address family: 

<pre style="font-size:12px">
vrf cust-2 rd 2:2
vrf cust-2 address-family ipv4 unicast import route-target 2:2
vrf cust-2 address-family ipv4 unicast export route-target 2:2
vrf cust-2 address-family ipv6 unicast import route-target 2:2
vrf cust-2 address-family ipv6 unicast export route-target 2:2
</pre>

After having created the VRF, we the interface and assign the interface to the VRF:

<pre style="font-size:12px">
interface GigabitEthernet0/0/0/2.2004 description c2-1
interface GigabitEthernet0/0/0/2.2004 vrf cust-2
interface GigabitEthernet0/0/0/2.2004 ipv4 address 10.0.0.17 255.255.255.252
interface GigabitEthernet0/0/0/2.2004 ipv6 address 2001:db8:1::9/127
interface GigabitEthernet0/0/0/2.2004 encapsulation dot1q 2004
</pre>

A think worth pointing out before moving to the BGP section is that we must have a route-policy in place for the IOSXR BGP sessions to accept any route import or export. To this end, we create the following route-policy:

<pre style="font-size:12px">
route-policy ACCEPT
  pass
end-policy
</pre>

Next is the actual BGP session and the options. All the customer BGP configuration for the VRF happens in the ‘router bgp 1 vrf cust-2’ configuration stanza. We want the PE to advertise the PE-CPE link for both address families, so we instruct the device to advertise the connected subnets:

<pre style="font-size:12px">
router bgp 1 vrf cust-2 address-family ipv4 unicast redistribute connected
router bgp 1 vrf cust-2 address-family ipv6 unicast redistribute connected
</pre>

Then we configure the IPv4 BGP session. We configure the remote-as and the route-policy. Additionally, we enable soft-reconfiguration inbound. This way, we can inspect the received routes from the neighbor, a behavior that is on by default on a Juniper device. To do this for both IPv4 as well as IPv6, we configure the following:

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



It is possible to explicitly configure an import and an export policy in Juniper but another option is the configuration of ‘vrf-target’. This will ensure the referenced route-target is attached to IPv4 as well as IPv6 routes. Additionally, it will make sure that routes tagged with that target are imported into the vrf as well. After configuring ‘vrf-target’, Juniper will actually create some ‘hidden’ route policies:

<pre style="font-size:12px">
salt@vmx01:r6> show policy    
Configured policies:
__vrf-export-cust-1-internal__
__vrf-export-cust-2-internal__
__vrf-import-cust-1-internal__
__vrf-import-cust-2-internal__
lbpp

salt@vmx01:r6> show policy __vrf-export-cust-2-internal__ 
Policy __vrf-export-cust-2-internal__:
    Term unnamed:
        then community + __vrf-community-cust-2-common-internal__ [target:2:2 ] accept

salt@vmx01:r6> show policy __vrf-import-cust-2-internal__    
Policy __vrf-import-cust-2-internal__:
    Term unnamed:
        from community __vrf-community-cust-2-common-internal__ [target:2:2 ]
        then accept
    Term unnamed:
        then reject

salt@vmx01:r6>
</pre>

Another thing to note here is that BGP would normally not advertise the connected routes. But since we have ‘vrf-target’ and ‘vrf-table-label’ configured under the instance stanza, the PE will be advertising the connected route.

In the Cisco IOSXR, we configured the import and export for the route-target under the address family in the VRF. Instead of specifying a route-target like this, it would have also been possible to configure a route-policy. This is something worth considering if you are creating a more complex VPN topology, a hub-and-spoke VPN for instance.
