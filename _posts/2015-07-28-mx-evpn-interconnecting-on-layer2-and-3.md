---
layout: post
title: EVPN on Juniper MX and interconnecting Data Centers on layer 2 and 3
tags: [juniper]
image: /img/juniper_logo.jpg
---

 
<p>
    After creating a single-homed layer 2 EVPN <a href="https://saidvandeklundert.net/2015-07-26-evpn-basics/" target="_blank">here</a>, let’s add some layer 3 routing and see in what way EVPN can benefit the datacenter.
</p>
<p>
    But first, have a look at a situation wherein a VPLS is connecting two data centers together:
</p>

![EVPN](/img/evpn_l2_l3_1.png "EVPN") 

<p>
    In the picture above, a VPLS exists between two bottom routers. By placing an IRB interface configured with VRRP in the VPLS, the MX can offer a redundant default gateway to the hosts in both datacenters. 
</p>
<p>
    For both inbound and outbound traffic, this may not provide for the most optimal traffic flow. Since the router in datacenter 1 is the VRRP master, the VM in datacenter 2 will send all outbound traffic via this router. This is putting a load on the links between the datacenters. 
</p>
<p>
    A similar problem exists with inbound traffic. If the router in datacenter 1 announces the entire subnet with a higher local preference for instance, inbound traffic towards the VM in datacenter 2 will also pass the router in datacenter 1.
</p>
<p>
    By using EVPN, we can optimize both inbound and outbound routing. To demonstrate this, I created the following setup:
</p>

![EVPN](/img/evpn_l2_l3_2.png "EVPN") 

<p>
    The above diagram shows that VMs can communicate with each other on layer 2 by using the EVPN. This can be achieved by using a VPLS (or a regular VLAN) as well. But with EVPN, the VMs can be made to use the layer 3 gateway of the nearest router. In fact, the gateway address (both layer 2 and layer 3) can be configured to be the same in both datacenters, skipping VRRP al together. 
</p>
<p>
    Whenever a VM or a host becomes active in the EVPN, the router in that particular datacenter becomes aware of that fact. 
	It will install a /32 route towards that VM or host. Using BGP, the router can advertise this route to the rest of the network. 
	By advertising the routes for all the hosts, EVPN can provide for more optimal routing compared to the VPLS + VRRP setup.
</p>
<p>
    So how can we get there?
</p>
<p>
    First we create a layer 2 EVPN between the 2 datacenters.  This configuration is still the same as it was in a previous post. The following is the configuration is applied to the MX480:
</p>

<pre style="font-size:12px">
set interfaces ge-0/1/0 flexible-vlan-tagging
set interfaces ge-0/1/0 encapsulation flexible-ethernet-services
set interfaces ge-0/1/0 unit 50 description CPE_EVPN
set interfaces ge-0/1/0 unit 50 encapsulation vlan-bridge
set interfaces ge-0/1/0 unit 50 vlan-id 50
set interfaces ge-0/1/0 unit 50 family bridge

set protocols bgp group rr family evpn signaling

set routing-instances evpn_vlan-based instance-type evpn
set routing-instances evpn_vlan-based vlan-id 50
set routing-instances evpn_vlan-based interface ge-0/1/0.50
set routing-instances evpn_vlan-based route-distinguisher 3:3
set routing-instances evpn_vlan-based vrf-target target:3:3
set routing-instances evpn_vlan-based protocols evpn</pre>  
<p>
    Adding a layer 3 gateway to this configuration is pretty straightforward:
</p>
<pre style="font-size:12px">
set routing-instances evpn_vlan-based routing-interface irb.50

set interfaces irb unit 50 description EVPN-Gateway-datacenter-1
set interfaces irb unit 50 family inet address 8.0.0.1/24</pre> 
<p>
    This configuration will work when you apply it to 1 of the two routers active in the EVPN. But this will not get us where we want. Instead of activating VRRP to offer a redundant gateway, we want to offer the exact same gateway in both datacenter 1 and 2. To be able to do this, we need to add two more configuration commands:
</p>
<pre style="font-size:12px">
set interfaces irb unit 50 mac 00:00:00:00:00:33
set routing-instances evpn_vlan-based protocols evpn default-gateway do-not-advertise</pre> 
<p>
    By setting the MAC address for the IRB interface, we can make sure that both of the routers that offer a gateway in the EVPN are reachable through the same MAC address. The second configuration command will make sure that the routers do not advertise their IRB interface to other routers.
</p>
<p>
    After adding this configuration on both the MX480 as well as the MX104, the situation is as follows:
</p>

![EVPN](/img/evpn_l2_l3_3.png "EVPN") 
             
<p>
    Both VMs will be able to reach each other on their layer 2 address and they will be able to reach their nearest gateway. There is no connectivity to the rest of the network yet. Let’s have a look at the routes toward the VMs from the MX480: 
</p>
<pre style="font-size:12px">
play@MX480-TEST-RE0> show route protocol evpn table evpn_vlan-based.evpn.0

evpn_vlan-based.evpn.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

2:3:3::50::88:e0:f3:55:f8:00/304
                   *[EVPN/170] 00:02:18
                      Indirect
2:3:3::50::88:e0:f3:55:f8:00::<font color='red'>8.0.0.2</font>/304
                   *[EVPN/170] 00:02:18
                      Indirect
3:3:3::50::1.1.1.21/304
                   *[EVPN/170] 2d 03:29:10
                      Indirect</pre>  
<p>
    In the EVPN routing table, there are routes to the VM in both datacenters. The route towards the VM in datacenter 1 not only shows a MAC address, it also displays the IP address of that VM. This information is also advertised towards the route-reflector (that I forgot to mention before):
</p>
<pre style="font-size:12px">
play@MX480-TEST-RE0> show route advertising-protocol bgp 1.1.1.1 table evpn_vlan-based.evpn.0

evpn_vlan-based.evpn.0: 6 destinations, 6 routes (6 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  2:3:3::50::88:e0:f3:55:f8:00/304
*                         Self                         100        I
  2:3:3::50::88:e0:f3:55:f8:00::8.0.0.2/304
*                         Self                         100        I
  3:3:3::50::1.1.1.21/304
*                         Self                         100        I</pre>   
<p>
    Since the EVPN is only active on two routers, the rest of the network will have no knowledge of these EVPN routes. In order to understand how we can distribute the route to all routers in the network, let’s look at what happened in the inet.0 table:
</p>
<pre style="font-size:12px">
play@MX480-TEST-RE0> show route protocol evpn table inet.0 output interface irb.50

inet.0: 42 destinations, 43 routes (38 active, 0 holddown, 4 hidden)
+ = Active Route, - = Last Active, * = Both

<font color='red'>8.0.0.2/32</font>         *[<font color='red'>EVPN/7</font>] 00:05:11
                    > via irb.50
{master}</pre>     
<p>
    The route towards the local VM is placed in the inet.0 routing table as being reachable via the irb interface. The route is created by the MX as soon as it learns of the existence of a particular device. For instance, as soon as I add another device using 8.0.0.200 in datacenter 1, the MX immediately creates a second /32 route for this device:
</p>

<pre style="font-size:12px">
play@MX480-TEST-RE0> show route protocol evpn table inet.0 output interface irb.50

inet.0: 42 destinations, 43 routes (38 active, 0 holddown, 4 hidden)
+ = Active Route, - = Last Active, * = Both

8.0.0.2/32         *[EVPN/7] 00:05:11
                    > via irb.50
<font color='red'>8.0.0.200/32</font>       *[EVPN/7] 00:06:09
                    > via irb.50

{master}</pre>         
<p>
    You can advertise these routes to the rest of the network by using a policy that matches these routes. In this example, I used the following policy:                    
</p>                
<pre style="font-size:12px">
set policy-options policy-statement bgp-export term local-evpn-routes <font color='red'>from protocol evpn</font>
set policy-options policy-statement bgp-export term local-evpn-routes then accept

set policy-options policy-statement bgp-export term evpn <font color='red'>from family evpn</font>
set policy-options policy-statement bgp-export term evpn then accept</pre>                  
<p>
    The first term will match the routes from protocol EVPN in the inet.0 table. The second term will allow the advertisement of the EVPN route in the EVPN table.  After the creation of this policy, it needs to be applied as an export policy and the BGP session needs to be configured for the ‘inet unicast’ family as well:
</p>
<pre style="font-size:12px">
set protocols bgp group rr family inet unicast
set protocols bgp group rr export bgp-export</pre>                  
<p>
    As soon as this is activated on the MX routers, they will start advertising hosts connected in their datacenter as /32 routes to the rest of the network. An example printout from the MX480:
</p>
<pre style="font-size:12px">
play@MX480-TEST-RE0> show route advertising-protocol bgp 1.1.1.1 table inet.0

inet.0: 42 destinations, 43 routes (38 active, 0 holddown, 4 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 8.0.0.2/32              Self                         100        I
* 8.0.0.200/32            Self                         100        I

play@MX480-TEST-RE0> show route receive-protocol bgp 1.1.1.1 table inet.0

inet.0: 42 destinations, 43 routes (38 active, 0 holddown, 4 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  8.0.0.3/32              1.1.1.20                     100        I</pre>    
<p>
    The traffic to or from a VM (or another device type) will follow the optimal path. When we look at the table from another router in the network, we can observe the following:
</p>
<pre style="font-size:12px">
play@Tiberius> show route 8.0.0.2

inet.0: 41 destinations, 41 routes (39 active, 0 holddown, 2 hidden)
+ = Active Route, - = Last Active, * = Both

<font color='red'>8.0.0.2/32</font>         *[BGP/170] 00:00:32, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    > to 2.0.0.33 via xe-0/2/0.9, label-switched-path <font color='red'>to_MX480</font>
{master}
play@Tiberius> show route 8.0.0.3

inet.0: 41 destinations, 41 routes (39 active, 0 holddown, 2 hidden)
+ = Active Route, - = Last Active, * = Both

<font color='red'>8.0.0.3/32</font>         *[BGP/170] 01:41:27, localpref 100, from 1.1.1.1
                      AS path: I, validation-state: unverified
                    > to 2.0.0.33 via xe-0/2/0.9, label-switched-path <font color='red'>to_MX104</font></pre>
<p>
    Since an MX can consult the inet.3 table for prefixes learned via BGP, the /32 routes that are advertised are installed in the forwarding table using the LSP towards the originator of the route as a next-hop. 
</p>
<p>
    Apart from optimal routing, another benefit is that VMs who are migrated can continue to use the same MAC address to forward traffic towards their new gateway. Their arp-cache will not have to time out or be cleared manually.
</p>
<p>
    Note, I placed the irb in the global table, but you can just as easy put it into a routing-instance.
</p>
<p>
    Really starting to like this EVPN by now!
    To wrap things up, the relevant configuration for the MX routers is the following: 
</p> 

<b>
MX480:
</b>

<pre style="font-size:12px">
set chassis network-services enhanced-ip

set interfaces ge-0/1/0 flexible-vlan-tagging
set interfaces ge-0/1/0 encapsulation flexible-ethernet-services
set interfaces ge-0/1/0 unit 50 description CPE_EVPN
set interfaces ge-0/1/0 unit 50 encapsulation vlan-bridge
set interfaces ge-0/1/0 unit 50 vlan-id 50
set interfaces ge-0/1/0 unit 50 family bridge

set interfaces irb unit 50 description EVPN-Gateway-datacenter-2
set interfaces irb unit 50 family inet address 8.0.0.1/24
set interfaces irb unit 50 mac 00:00:00:00:00:33

set routing-options autonomous-system 1
set routing-options forwarding-table chained-composite-next-hop ingress evpn

set protocols bgp local-address 1.1.1.21
set protocols bgp out-delay 2
set protocols bgp log-updown
set protocols bgp group rr type internal
set protocols bgp group rr family inet unicast
set protocols bgp group rr family evpn signaling
set protocols bgp group rr authentication-key "$9$CVAyuOIyrKvWXVwGjHmQz"
set protocols bgp group rr export bgp-export
set protocols bgp group rr peer-as 1
set protocols bgp group rr neighbor 1.1.1.1 description Aurelius_rr

set routing-instances evpn_vlan-based instance-type evpn
set routing-instances evpn_vlan-based vlan-id 50
set routing-instances evpn_vlan-based interface ge-0/1/0.50
set routing-instances evpn_vlan-based routing-interface irb.50
set routing-instances evpn_vlan-based route-distinguisher 3:3
set routing-instances evpn_vlan-based vrf-target target:3:3
set routing-instances evpn_vlan-based protocols evpn default-gateway do-not-advertise
                
set policy-options policy-statement bgp-export term local-evpn-routes from protocol evpn
set policy-options policy-statement bgp-export term local-evpn-routes then accept

set policy-options policy-statement bgp-export term evpn from family evpn
set policy-options policy-statement bgp-export term evpn then accept
</pre>    
<b>
MX104:
</b>
<pre style="font-size:12px">
set chassis network-services enhanced-ip

set interfaces xe-2/0/0 flexible-vlan-tagging
set interfaces xe-2/0/0 encapsulation flexible-ethernet-services
set interfaces xe-2/0/0 unit 50 description CPE_EVPN
set interfaces xe-2/0/0 unit 50 encapsulation vlan-bridge
set interfaces xe-2/0/0 unit 50 vlan-id 50
set interfaces xe-2/0/0 unit 50 family bridge

set interfaces irb unit 50 description EVPN-Gateway-datacenter-2
set interfaces irb unit 50 family inet address 8.0.0.1/24
set interfaces irb unit 50 mac 00:00:00:00:00:33

set routing-options autonomous-system 1
set routing-options forwarding-table chained-composite-next-hop ingress evpn

set protocols bgp local-address 1.1.1.20
set protocols bgp out-delay 2
set protocols bgp log-updown
set protocols bgp group rr type internal
set protocols bgp group rr family inet unicast
set protocols bgp group rr family evpn signaling
set protocols bgp group rr authentication-key "$9$CVAyuOIyrKvWXVwGjHmQz"
set protocols bgp group rr export bgp-export
set protocols bgp group rr peer-as 1
set protocols bgp group rr neighbor 1.1.1.1 description Aurelius_rr

set routing-instances evpn_vlan-based instance-type evpn
set routing-instances evpn_vlan-based vlan-id 50
set routing-instances evpn_vlan-based interface xe-2/0/0.50
set routing-instances evpn_vlan-based routing-interface irb.50
set routing-instances evpn_vlan-based route-distinguisher 3:3
set routing-instances evpn_vlan-based vrf-target target:3:3
set routing-instances evpn_vlan-based protocols evpn default-gateway do-not-advertise
                
set policy-options policy-statement bgp-export term local-evpn-routes from protocol evpn
set policy-options policy-statement bgp-export term local-evpn-routes then accept

set policy-options policy-statement bgp-export term evpn from family evpn
set policy-options policy-statement bgp-export term evpn then accept
</pre>   