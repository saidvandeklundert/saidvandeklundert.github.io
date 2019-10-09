---
layout: post
title: Juniper MPLS VPN OSPF sham-link
tags: [juniper, ospf, mpls, vpn]
image: /img/juniper_logo.jpg
---

<p>
    This is an example on using the OSPF sham link in a BGP signaled MPLS VPN. The scenario is as follows:
</p>


![OSPF sham link](/img/ospf_sham_link_1.png "OSPF sham link")    


<p>
    In the scenario above, you can see two PE routers.
    These are Juno and Ceres. Both PE routers are running MP-BGP to exchange vpn routes with each other. 
    They are also running OSPF with the CPE’s. The PE is redistributing BGP routes into OSPF.
</p>
<p>
    The two CPE’s are Luna and Orcus. They do not only run OSPF with the PE they are connected to, they also run OSPF with each other. 
    The configuration is as follows:
</p>
<b>
    Juno:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 105 description LUNA
set interfaces xe-2/0/1 unit 105 vlan-id 105
set interfaces xe-2/0/1 unit 105 family inet mtu 1500
set interfaces xe-2/0/1 unit 105 family inet address 4.0.0.22/30
set interfaces lo0 unit 50 description ipvpn-Juno
set interfaces lo0 unit 50 family inet address 10.0.0.5/32

set policy-options policy-statement bgp-2-ospf term 1 from protocol bgp
set policy-options policy-statement bgp-2-ospf term 1 then accept
set policy-options policy-statement bgp-2-ospf term reject-all then reject

set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.105
set routing-instances ipvpn interface lo0.50
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn protocols ospf export bgp-2-ospf
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface xe-2/0/1.105 interface-type p2p
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface lo0.50
</pre>                    
<b>
    Ceres:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 111 description ORCUS
set interfaces xe-2/0/1 unit 111 vlan-id 111
set interfaces xe-2/0/1 unit 111 family inet mtu 1500
set interfaces xe-2/0/1 unit 111 family inet address 4.0.0.46/30
set interfaces lo0 unit 20 description ipvpn-Ceres
set interfaces lo0 unit 20 family inet address 10.0.0.2/32

set policy-options policy-statement bgp-2-ospf term 1 from protocol bgp
set policy-options policy-statement bgp-2-ospf term 1 then accept
set policy-options policy-statement bgp-2-ospf term reject-all then reject

set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.111
set routing-instances ipvpn interface lo0.20
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn protocols ospf export bgp-2-ospf
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface xe-2/0/1.111 interface-type p2p
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface lo0.20
</pre>    
<b>
    Luna:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 150 description Orcus
set interfaces xe-2/0/1 unit 150 vlan-id 150
set interfaces xe-2/0/1 unit 150 family inet mtu 1500
set interfaces xe-2/0/1 unit 150 family inet address 192.168.1.3/24
set interfaces xe-2/0/0 unit 105 description JUNO
set interfaces xe-2/0/0 unit 105 vlan-id 105
set interfaces xe-2/0/0 unit 105 family inet mtu 1500
set interfaces xe-2/0/0 unit 105 family inet address 4.0.0.21/30
set interfaces lo0 unit 8 description LUNA
set interfaces lo0 unit 8 family inet address 10.0.0.8/32

set protocols ospf area 0.0.0.0 interface xe-2/0/0.105 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.8 interface-type p2p
set protocols ospf area 0.0.0.0 interface xe-2/0/1.150
</pre>    
<b>
    Orcus:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 111 description CERES
set interfaces xe-2/0/0 unit 111 vlan-id 111
set interfaces xe-2/0/0 unit 111 family inet mtu 1500
set interfaces xe-2/0/0 unit 111 family inet address 4.0.0.45/30
set interfaces xe-2/0/0 unit 111 family iso
set interfaces xe-2/0/0 unit 150 description Luna
set interfaces xe-2/0/0 unit 150 vlan-id 150
set interfaces xe-2/0/0 unit 150 family inet mtu 1500
set interfaces xe-2/0/0 unit 150 family inet address 192.168.1.2/24 
set interfaces lo0 unit 11 family inet address 10.0.0.11/32

set protocols ospf area 0.0.0.0 interface lo0.11
set protocols ospf area 0.0.0.0 interface xe-2/0/0.111 interface-type p2p
set protocols ospf area 0.0.0.0 interface xe-2/0/0.150
</pre>    
<p>
    Without the use of a sham link, the CPE’s will prefer to use the link that connects them directly together when they need to send traffic to subnets advertised by either of them. 
</p>                    

![OSPF sham link](/img/ospf_sham_link_traffic_1.png "OSPF sham link")  

                   
<p>
    The following is a printout from Luna. 
</p>
<pre style="font-size:12px"> 
play@MX104-TEST-HB:Luna> show route 10.0.0.11

inet.0: 19 destinations, 20 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[OSPF/10] 00:01:57, metric 3
                    > to 192.168.1.2 via xe-2/0/1.150

play@MX104-TEST-HB:Luna> show ospf route 10.0.0.11
Topology default Route Table:

Prefix             Path  Route      NH       Metric NextHop       Nexthop
                   Type  Type       Type            Interface     Address/LSP
10.0.0.11          <font color='red'>Intra</font> Router     IP            3 xe-2/0/1.150  192.168.1.2
10.0.0.11/32       <font color='red'>Intra</font> Network    IP            3 xe-2/0/1.150  192.168.1.2
</pre>    
<p>
    This printout show that Luna will use the link it has to Orcus for all traffic destined to that router.  Luna is using an intra area OSPF route.
</p>
<p>
    Juno is also advertising the route towards Luna, but the route coming from Juno is not used. 
    This has nothing to do with the metric that is associated with the route. 
    When we deactivate OSPF on the interfaces between Orcus and Luna, we can see why the route via Juno is less attractive:
</p>

<pre style="font-size:12px">
play@MX104-TEST-HB:Luna> show route 10.0.0.11

inet.0: 17 destinations, 19 routes (17 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[OSPF/10] 00:01:45, metric 2
                    > to 4.0.0.22 via xe-2/0/0.105

play@MX104-TEST-HB:Luna> show ospf route 10.0.0.11
Topology default Route Table:

Prefix             Path  Route      NH       Metric NextHop       Nexthop
                   Type  Type       Type            Interface     Address/LSP
10.0.0.11/32       <font color='red'>Inter</font> Network    IP            2 xe-2/0/0.105  4.0.0.22
</pre>   
<p>
    As soon as the OSPF adjacency between Luna an Orcus is lost, the route via Juno is installed (the next-hop changed from 192.168.1.2 to 4.0.0.22).  
    As you can see, this route is an inter area OSPF route. 
</p>
<p>
    In OSPF, intra area routes are routes that are originating from within the same area. 
    This is the case for routes learned from Orcus. Inter area routes are routes that originated outside the area and were advertised into the area. 
</p>
<p>
    In our case, the fact that Ceres and Juno are running OSPF in area 0 is not relevant. 
    The PE’s use BGP to update each other with this routing information, so Juno learned the route via MP-BGP and redistributed the route into OSPF. 
    This makes it an inter area route. OSPF will always prefer an intra area route over an inter area route, this is regardless of the metric that is associated with that route. 
</p>
<p>
    The sham link is an unnumbered point-to-point link inside a routing-instance between two PE routers. Across the sham link, the PE routers can build an OSPF adjacency directly with each other. Because they can build the OSPF adjacency directly with each other, the routes exchanged between the PE's will remain intra area routes. This as opposed to having to redistribute routing information from BGP into OSPF. 
    Thus, by using the sham link, we can make the CPE’s use the link they have with the PE routers and make them send all traffic through the MPLS VPN.
</p>


![OSPF sham link](/img/ospf_sham_link_2.png "OSPF sham link")  

<p>
    To configure the sham link in our scenario, we need to add the following configuration:
</p>
<b>
    Juno:
</b>
<pre style="font-size:12px">
set routing-instances ipvpn protocols ospf sham-link local 10.0.0.5
set routing-instances ipvpn protocols ospf area 0.0.0.0 sham-link-remote 10.0.0.2 metric 1
</pre>                     
<b>
    Ceres:
</b>
<pre style="font-size:12px">
set routing-instances ipvpn protocols ospf sham-link local 10.0.0.2
set routing-instances ipvpn protocols ospf area 0.0.0.0 sham-link-remote 10.0.0.5 metric 1
</pre>  
<p>
    After the configuration of this sham link, we can use the following commands to verify our configuration:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Juno> show ospf neighbor instance ipvpn
Address          Interface              State     ID               Pri  Dead
10.0.0.2         shamlink.0             Full      10.0.0.2           0    34
4.0.0.21         xe-2/0/1.105           Full      10.0.0.8         128    39

play@MX104-TEST-HB:Juno> show ospf interface shamlink.0 extensive instance ipvpn
Interface           State   Area            DR ID           BDR ID          Nbrs
shamlink.0          PtToPt  0.0.0.0         0.0.0.0         0.0.0.0            1
  Type: P2P, Address: 0.0.0.0, Mask: 0.0.0.0, MTU: 0, Cost: 1
  Local: 10.0.0.5, Remote: 10.0.0.2
  Adj count: 1
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  Auth type: None
  Protection type: None, No eligible backup
  Topology default (ID 0) -> Cost: 1
</pre>              
<p>
    This is a direct neighbor relationship in area 0.0.0.0. When we move to Luna, we can see that the route learned via Juno is now an intra area route:
</p>

<pre style="font-size:12px">
play@MX104-TEST-HB:Luna> show ospf route 10.0.0.11
Topology default Route Table:

Prefix             Path  Route      NH       Metric NextHop       Nexthop
                   Type  Type       Type            Interface     Address/LSP
10.0.0.11          Intra Router     IP            3 xe-2/0/1.150  192.168.1.2
                                                    xe-2/0/0.105  4.0.0.22
10.0.0.11/32       Intra Network    IP            3 xe-2/0/1.150  192.168.1.2
                                                    xe-2/0/0.105  4.0.0.22
</pre>    
<p>
    Both routes are now intra area routes and they are contending based on their metric. 
    When we increase the cost of the direct link to 4, we can see the following:
</p>                
<pre style="font-size:12px">
play@MX104-TEST-HB:Luna> configure
Entering configuration mode

[edit]
play@MX104-TEST-HB:Luna# set protocols ospf area 0.0.0.0 interface xe-2/0/1.150 metric 4

[edit]
play@MX104-TEST-HB:Luna# commit and-quit
commit complete
Exiting configuration mode

play@MX104-TEST-HB:Luna> show route 10.0.0.11

inet.0: 19 destinations, 21 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.11/32       *[OSPF/10] 00:00:07, metric 3
                    > to 4.0.0.22 via xe-2/0/0.105
</pre>                   
<p>
    Let’s apply the same metric for the link on the Orcus router:
</p>                    
<pre style="font-size:12px">
play@MX104-TEST-HB:Orcus> configure
Entering configuration mode

[edit]
play@MX104-TEST-HB:Orcus# set protocols ospf area 0.0.0.0 interface xe-2/0/0.150 metric 4

[edit]
play@MX104-TEST-HB:Orcus# commit and-quit
show rou        10.0.0.8
commit complete
Exiting configuration mode

play@MX104-TEST-HB:Orcus> show route 10.0.0.8

inet.0: 20 destinations, 20 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.8/32        *[OSPF/10] 00:00:04, metric 3
                    > to 4.0.0.46 via xe-2/0/0.111 
</pre>             
<p>
    Instead of using the link with each other, traffic will now be sent across the MPLS VPN. 
</p>

![OSPF sham link](/img/ospf_sham_link_traffic_2.png "OSPF sham link")  

<p>
    The relevant configuration with the sham link:
</p>
<b>
    Juno:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 105 description LUNA
set interfaces xe-2/0/1 unit 105 vlan-id 105
set interfaces xe-2/0/1 unit 105 family inet mtu 1500
set interfaces xe-2/0/1 unit 105 family inet address 4.0.0.22/30
set interfaces lo0 unit 50 description ipvpn-Juno
set interfaces lo0 unit 50 family inet address 10.0.0.5/32
set policy-options policy-statement bgp-2-ospf term 1 from protocol bgp
set policy-options policy-statement bgp-2-ospf term 1 then accept
set policy-options policy-statement bgp-2-ospf term reject-all then reject
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.105
set routing-instances ipvpn interface lo0.50
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn protocols ospf export bgp-2-ospf
set routing-instances ipvpn protocols ospf sham-link local 10.0.0.5
set routing-instances ipvpn protocols ospf area 0.0.0.0 sham-link-remote 10.0.0.2 metric 1
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface xe-2/0/1.105 interface-type p2p
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface lo0.50
</pre> 
<b>
    Ceres:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 111 description ORCUS
set interfaces xe-2/0/1 unit 111 vlan-id 111
set interfaces xe-2/0/1 unit 111 family inet mtu 1500
set interfaces xe-2/0/1 unit 111 family inet address 4.0.0.46/30
set interfaces lo0 unit 20 description ipvpn-Ceres
set interfaces lo0 unit 20 family inet address 10.0.0.2/32
set policy-options policy-statement bgp-2-ospf term 1 from protocol bgp
set policy-options policy-statement bgp-2-ospf term 1 then accept
set policy-options policy-statement bgp-2-ospf term reject-all then reject
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.111
set routing-instances ipvpn interface lo0.20
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn protocols ospf export bgp-2-ospf
set routing-instances ipvpn protocols ospf sham-link local 10.0.0.2
set routing-instances ipvpn protocols ospf area 0.0.0.0 sham-link-remote 10.0.0.5 metric 1
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface xe-2/0/1.111 interface-type p2p
set routing-instances ipvpn protocols ospf area 0.0.0.0 interface lo0.20
</pre> 
<b>
    Luna:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 150 description Orcus
set interfaces xe-2/0/1 unit 150 vlan-id 150
set interfaces xe-2/0/1 unit 150 family inet mtu 1500
set interfaces xe-2/0/1 unit 150 family inet address 192.168.1.3/24
set interfaces xe-2/0/0 unit 105 description JUNO
set interfaces xe-2/0/0 unit 105 vlan-id 105
set interfaces xe-2/0/0 unit 105 family inet mtu 1500
set interfaces xe-2/0/0 unit 105 family inet address 4.0.0.21/30
set interfaces lo0 unit 8 description LUNA
set interfaces lo0 unit 8 family inet address 10.0.0.8/32
set protocols ospf area 0.0.0.0 interface xe-2/0/0.105 interface-type p2p
set protocols ospf area 0.0.0.0 interface lo0.8 interface-type p2p
set protocols ospf area 0.0.0.0 interface xe-2/0/1.150 metric 4
</pre> 
<b>
    Orcus:
</b>
<pre style="font-size:12px"> 
set interfaces xe-2/0/0 unit 111 description CERES
set interfaces xe-2/0/0 unit 111 vlan-id 111
set interfaces xe-2/0/0 unit 111 family inet mtu 1500
set interfaces xe-2/0/0 unit 111 family inet address 4.0.0.45/30
set interfaces xe-2/0/0 unit 111 family iso
set interfaces xe-2/0/0 unit 150 description Luna
set interfaces xe-2/0/0 unit 150 vlan-id 150
set interfaces xe-2/0/0 unit 150 family inet mtu 1500
set interfaces xe-2/0/0 unit 150 family inet address 192.168.1.2/24 
set interfaces lo0 unit 11 family inet address 10.0.0.11/32
set protocols ospf area 0.0.0.0 interface lo0.11
set protocols ospf area 0.0.0.0 interface xe-2/0/0.111 interface-type p2p
set protocols ospf area 0.0.0.0 interface xe-2/0/0.150 metric 4
</pre>                 

