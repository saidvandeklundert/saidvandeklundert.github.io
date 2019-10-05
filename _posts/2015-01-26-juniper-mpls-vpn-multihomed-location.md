---
layout: post
title: Juniper multihomed IP VPN location.
image: /img/juniper_logo.jpg
---

<p>
    This article offers some insight into how you could decide to build a multihomed Layer 3 IP VPN or Layer 3 MPLS VPN. First I’ll go over the topology. 
    After this, you will find the PE and CPE configuration. I’ll end with some verification and show commands. 
</p>         
<br>
<br>
<p>
    <h4>
        The topology:
    </h4>
</p>                             

![MPLS VPN](/img/ipvpn_multihomed_example_1.png "MPLS VPN topology") 

<p>
    There are three PE routers. These are Liber, Juno and Ceres. 
    The three PE routers are connected together via an MPLS network and have BGP configured to exchange routing information for the IP VPN called ‘ipvpn’.
</p>
<p>
    There are two locations connected to the IP VPN. 
    On one location, Genius is the CPE. 
    This CPE has a connection to Liber. 
    On Liber, a static route is configured with the prefixes in use on Genius. This static route is being redistributed into BGP. 
</p>
<p>
    Two other CPE’s connect the multihomed location to the IP VPN. 
    These two CPE's are Luna and Orcus. These two CPE’s are offering the customer line- and node-redundancy. This is achieved by using both BGP and VRRP. 
    In order to exchange routing information, the CPE’s run BGP with each other and with the PE’s they are connected to (Juno and Ceres).
    Luna and Orcus also run VRRP to provide a single gateway to the customer’s device. 
    The customers device is named Saturn. 
</p>        

![MPLS VPN](/img/ipvpn_multihomed_example_2.png "MPLS VPN") 

<p>
    The prefix in which the VRRP address is configured as well as the CPE’s loopback IP addresses are made redundant across the EBGP sessions in use between the CPE’s and the PE’s. 
    Luna is acting as the primary CPE for all these prefixes and Orcus is acting as Luna’s backup. 
</p>
<p>
    To make sure all traffic from the IP VPN to the location uses the connection Juno-Luna, the local-preference BGP attribute is used. 
    The local-preference attribute has the highest value on Juno, making sure that all other PE’s will prefer to install that route into their routing table as opposed to the Ceres – Orcus connection.
</p>

              
![MPLS VPN](/img/ipvpn_multihomed_loc_pref.png "MPLS VPN") 

<p>
    To make sure all traffic from the customer location into the IP VPN also uses the Juno-Luna connection, the MED BGP attribute is used. 
    The MED value Juno is applying to the routes is 10. 
    This is lower than the value Ceres applies to the routes, which is 150. 
    Since Luna and Orcus are running IBGP between them, they will exchange all the routes received from the PE’s. 
    The BGP route selection will favor the route with the lowest MED. 
    Therefore, as long as the Juno-Luna BGP session is up, routes received from Juno will be favored over the routes received by Ceres.
</p>

![MPLS VPN](/img/ipvpn_multihomed_med.png "MPLS VPN") 
                
<p>
    The CPE’s also apply a community to all the advertised routes. 
    This will prevent the PE’s from advertising CPE-learned routes back to the CPE’s. 
    The ‘next-hop-self’ that is applied in the policy configured between the CPE’s is needed because in IBGP, the next-hop attribute remains unaltered. 
    If the next-hop would not be changed, we would have to redistribute the prefixes in use between the PE and the CPE.
</p>
<p>
    <h4>
        The configuration:
    </h4>
</p>  
<b>
    PE Liber:
</b>
<pre style="font-size:12px">   
set interfaces xe-2/0/1 unit 124 description GENIUS
set interfaces xe-2/0/1 unit 124 vlan-id 124
set interfaces xe-2/0/1 unit 124 family inet mtu 1500
set interfaces xe-2/0/1 unit 124 family inet address 4.0.0.98/30
set interfaces xe-2/0/1 unit 124 family iso
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.124
set routing-instances ipvpn interface xe-2/0/1.125
set routing-instances ipvpn interface lo0.70
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn routing-options static route 10.0.0.3/32 next-hop 4.0.0.97
</pre>        
<b>
    PE Juno:
</b>
<pre style="font-size:12px">   
set interfaces xe-2/0/1 unit 105 description LUNA
set interfaces xe-2/0/1 unit 105 vlan-id 105
set interfaces xe-2/0/1 unit 105 family inet mtu 1500
set interfaces xe-2/0/1 unit 105 family inet address 4.0.0.22/30
set interfaces xe-2/0/1 unit 105 family iso
set policy-options policy-statement multihomed-ipvpn term no-cpe-originated from community cpe-originated-route
set policy-options policy-statement multihomed-ipvpn term no-cpe-originated then reject
set policy-options policy-statement multihomed-ipvpn term accept-rest then accept
set policy-options policy-statement multihomed-ipvpn-primary then local-preference 150
set policy-options policy-statement multihomed-ipvpn-primary then accept
set policy-options community cpe-originated-route members origin:1000:1000
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.105
set routing-instances ipvpn interface lo0.50
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn protocols bgp group CPE type external
set routing-instances ipvpn protocols bgp group CPE passive
set routing-instances ipvpn protocols bgp group CPE authentication-key "$9$tAvZO1EleMWL7wYDHqfF3"
set routing-instances ipvpn protocols bgp group CPE peer-as 65500
set routing-instances ipvpn protocols bgp group CPE as-override
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21 metric-out 10
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21 import multihomed-ipvpn-primary
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21 export multihomed-ipvpn
</pre>                   
<b>
    PE Ceres:
</b>
<pre style="font-size:12px">   
set interfaces xe-2/0/1 unit 111 description ORCUS
set interfaces xe-2/0/1 unit 111 vlan-id 111
set interfaces xe-2/0/1 unit 111 family inet mtu 1500
set interfaces xe-2/0/1 unit 111 family inet address 4.0.0.46/30
set interfaces xe-2/0/1 unit 111 family iso
set policy-options policy-statement multihomed-ipvpn term no-cpe-originated from community cpe-originated-route
set policy-options policy-statement multihomed-ipvpn term no-cpe-originated then reject
set policy-options policy-statement multihomed-ipvpn term accept-rest then accept
set policy-options policy-statement multihomed-ipvpn-secondary then local-preference 10
set policy-options policy-statement multihomed-ipvpn-secondary then accept
set policy-options community cpe-originated-route members origin:1000:1000
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.111
set routing-instances ipvpn interface lo0.20
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn protocols bgp group CPE type external
set routing-instances ipvpn protocols bgp group CPE passive
set routing-instances ipvpn protocols bgp group CPE authentication-key "$9$tAvZO1EleMWL7wYDHqfF3"
set routing-instances ipvpn protocols bgp group CPE peer-as 65500
set routing-instances ipvpn protocols bgp group CPE as-override
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.45 metric-out 150
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.45 import multihomed-ipvpn-secondary
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.45 export multihomed-ipvpn
</pre>      
<b>
    CPE Luna (primary):
</b>
<pre style="font-size:12px">   
set interfaces xe-2/0/0 unit 105 description JUNO
set interfaces xe-2/0/0 unit 105 vlan-id 105
set interfaces xe-2/0/0 unit 105 family inet mtu 1500
set interfaces xe-2/0/0 unit 105 family inet address 4.0.0.21/30
set interfaces xe-2/0/0 unit 105 family iso
set interfaces xe-2/0/1 unit 150 description Saturn_Orcus
set interfaces xe-2/0/1 unit 150 vlan-id 150
set interfaces xe-2/0/1 unit 150 family inet mtu 1500
set interfaces xe-2/0/1 unit 150 family inet address 192.168.1.3/24 vrrp-group 1 virtual-address 192.168.1.1
set interfaces xe-2/0/1 unit 150 family inet address 192.168.1.3/24 vrrp-group 1 priority 250
set interfaces lo0 unit 8 description LUNA
set interfaces lo0 unit 8 family inet address 10.0.0.8/32
set protocols bgp group pe-connection type external
set protocols bgp group pe-connection hold-time 10
set protocols bgp group pe-connection log-updown
set protocols bgp group pe-connection authentication-key "$9$gPJGjmPTQF6p0yevL-d"
set protocols bgp group pe-connection export bgp-export
set protocols bgp group pe-connection peer-as 1
set protocols bgp group pe-connection neighbor 4.0.0.22 description Ceres
set protocols bgp group cpe-connection type internal
set protocols bgp group cpe-connection hold-time 10
set protocols bgp group cpe-connection authentication-key "$9$ty05O1EleMWL7wYDHqfF3"
set protocols bgp group cpe-connection export cpe-connection
set protocols bgp group cpe-connection peer-as 65500
set protocols bgp group cpe-connection neighbor 192.168.1.2 description Orcus
set policy-options prefix-list direct-lo0 apply-path "logical-systems Luna interfaces lo0 unit <*> family inet address <*>"
set policy-options policy-statement bgp-export term direct from protocol direct
set policy-options policy-statement bgp-export term direct from prefix-list direct-lo0
set policy-options policy-statement bgp-export term direct then community add cpe-originated-route
set policy-options policy-statement bgp-export term direct then accept
set policy-options policy-statement bgp-export term Saturn-prefix from protocol direct
set policy-options policy-statement bgp-export term Saturn-prefix from route-filter 192.168.1.0/24 exact
set policy-options policy-statement bgp-export term Saturn-prefix then community add cpe-originated-route
set policy-options policy-statement bgp-export term Saturn-prefix then accept
set policy-options policy-statement bgp-export term Saturn-Lo0 from protocol static
set policy-options policy-statement bgp-export term Saturn-Lo0 then community add cpe-originated-route
set policy-options policy-statement bgp-export term Saturn-Lo0 then accept
set policy-options policy-statement cpe-connection term all then next-hop self
set policy-options community cpe-originated-route members origin:1000:1000
set routing-options static route 10.0.0.12/32 next-hop 192.168.1.254
</pre>  
<b>
   CPE Orcus (secondary): 
</b>
<pre style="font-size:12px">   
set interfaces xe-2/0/0 unit 111 description CERES
set interfaces xe-2/0/0 unit 111 vlan-id 111
set interfaces xe-2/0/0 unit 111 family inet mtu 1500
set interfaces xe-2/0/0 unit 111 family inet address 4.0.0.45/30
set interfaces xe-2/0/0 unit 111 family iso
set interfaces xe-2/0/0 unit 150 description Saturn_Luna
set interfaces xe-2/0/0 unit 150 vlan-id 150
set interfaces xe-2/0/0 unit 150 family inet mtu 1500
set interfaces xe-2/0/0 unit 150 family inet address 192.168.1.2/24 vrrp-group 1 virtual-address 192.168.1.1
set interfaces xe-2/0/0 unit 150 family inet address 192.168.1.2/24 vrrp-group 1 priority 150
set interfaces lo0 unit 11 description ORCUS
set interfaces lo0 unit 11 family inet address 10.0.0.11/32
set protocols bgp group pe-connection type external
set protocols bgp group pe-connection hold-time 10
set protocols bgp group pe-connection log-updown
set protocols bgp group pe-connection authentication-key "$9$gPJGjmPTQF6p0yevL-d"
set protocols bgp group pe-connection export bgp-export
set protocols bgp group pe-connection peer-as 1
set protocols bgp group pe-connection neighbor 4.0.0.46 description Ceres
set protocols bgp group cpe-connection type internal
set protocols bgp group cpe-connection hold-time 10
set protocols bgp group cpe-connection authentication-key "$9$ty05O1EleMWL7wYDHqfF3"
set protocols bgp group cpe-connection export cpe-connection
set protocols bgp group cpe-connection peer-as 65500
set protocols bgp group cpe-connection neighbor 192.168.1.3 description Luna
set policy-options prefix-list direct-lo0 apply-path "logical-systems Orcus interfaces lo0 unit <*> family inet address <*>"
set policy-options policy-statement bgp-export term direct from protocol direct
set policy-options policy-statement bgp-export term direct from prefix-list direct-lo0
set policy-options policy-statement bgp-export term direct then community add cpe-originated-route
set policy-options policy-statement bgp-export term direct then accept
set policy-options policy-statement bgp-export term Saturn-prefix from protocol direct
set policy-options policy-statement bgp-export term Saturn-prefix from route-filter 192.168.1.0/24 exact
set policy-options policy-statement bgp-export term Saturn-prefix then community add cpe-originated-route
set policy-options policy-statement bgp-export term Saturn-prefix then accept
set policy-options policy-statement bgp-export term Saturn-Lo0 from protocol static
set policy-options policy-statement bgp-export term Saturn-Lo0 then community add cpe-originated-route
set policy-options policy-statement bgp-export term Saturn-Lo0 then accept
set policy-options policy-statement cpe-connection term all then next-hop self
set policy-options community cpe-originated-route members origin:1000:1000
set routing-options static route 10.0.0.12/32 next-hop 192.168.1.254
set routing-options autonomous-system 65500 
</pre>                   
<br>
<br>
<p>
    <h4>
        The Verification:
    </h4>
</p>
<p>
    Traffic flow when both routers are up:
</p>           

![MPLS VPN](/img/ipvpn_multihomed_example_3.png "MPLS VPN") 

<p>
    Let’s start by verifying the end-to-end connectivity from the Saturn router:
</p>
<pre style="font-size:12px">   
play@MX104-TEST-HB:Saturn> ping 10.0.0.3 source 10.0.0.12
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=61 time=19.715 ms
64 bytes from 10.0.0.3: icmp_seq=1 ttl=61 time=0.648 ms
^C
--- 10.0.0.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.648/10.181/19.715/9.533 ms

play@MX104-TEST-HB:Saturn> traceroute 10.0.0.3 source 10.0.0.12
traceroute to 10.0.0.3 (10.0.0.3) from 10.0.0.12, 30 hops max, 40 byte packets
 1  192.168.1.3 (192.168.1.3)  0.577 ms  0.443 ms  0.374 ms
 2  <font color='red'>4.0.0.22</font> (4.0.0.22)  0.444 ms  0.422 ms  0.410 ms
 3  4.0.0.98 (4.0.0.98)  0.469 ms  0.458 ms  0.432 ms
 4  10.0.0.3 (10.0.0.3)  0.769 ms  0.605 ms  0.573 ms

play@MX104-TEST-HB:Saturn>
</pre>  
<p>
    The previous output shows us there is IP connectivity and that traffic passes the Juno PE (4.0.0.22).
</p>
<p>
    On to Luna. Here we can see that the VRRP state is master and that the route via Juno is used when traffic is sent from Saturn to Genius:
</p>                
<pre style="font-size:12px">   
play@MX104-TEST-HB:Luna> show vrrp
Interface     State       Group   VR state VR Mode   Timer    Type   Address
xe-2/0/1.150  up              1   <font color='red'>master</font>   Active      A  0.560 lcl    192.168.1.3
                                                                vip    192.168.1.1

play@MX104-TEST-HB:Luna> show route 10.0.0.3

inet.0: 19 destinations, 21 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.3/32        *[BGP/170] 1w1d 00:32:28, MED 10, localpref 100
                      AS path: 1 I, validation-state: unverified
                    > to <font color='red'>4.0.0.22</font> via xe-2/0/0.105
</pre>  
<p>
The CPE’s were not configured to advertise inactive routes. Therefore, we will only see the routes into the IPVPN through both PE’s on Orcus:
</p>
<pre style="font-size:12px">              
play@MX104-TEST-HB:Orcus> show route 10.0.0.3

inet.0: 19 destinations, 33 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.3/32        *[BGP/170] 00:00:22, <font color='red'>MED 10</font>, localpref 100
                      AS path: 1 I, validation-state: unverified
                    > to 192.168.1.3 via xe-2/0/0.150
                    [BGP/170] 1w1d 01:19:45, <font color='red'>MED 150</font>, localpref 100
                      AS path: 1 I, validation-state: unverified
                    > to 4.0.0.46 via xe-2/0/0.111
</pre>
<p>
    The previous output shows us that Orcus receives the route towards 10.0.0.3 from both Luna and the PE, which is Ceres. 
    Since the MED from the Ceres route is higher than the MED from the Luna route, the Ceres route is not advertised to Luna and remains inactive.
</p>
<p>
    On Liber, we can see that return traffic towards Saturn will run via Luna as well. This is where the local preference comes in:
</p>

<pre style="font-size:12px">>      
play@MX104-TEST-HB:Liber> show route table ipvpn.inet.0 10.0.0.12 active-path

ipvpn.inet.0: 20 destinations, 33 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.12/32       *[BGP/170] 1w1d 01:08:29, <font color='red'>localpref 150</font>, from <font color='red'>10.0.0.2</font> <- Ceres is a route-reflector
                      AS path: 65500 I, validation-state: unverified
                    > to 4.0.0.58 via xe-2/0/0.114, Push 16

play@MX104-TEST-HB:Liber> show route table ipvpn.inet.0 10.0.0.12 active-path detail

ipvpn.inet.0: 20 destinations, 33 routes (20 active, 0 holddown, 0 hidden)
10.0.0.12/32 (2 entries, 1 announced)
	.. output omitted ..        
                Originator ID: <font color='red'>10.0.0.5</font>
                Communities: target:1:1 origin:1000:1000
                Import Accepted
                VPN Label: 16
                Localpref: 150
	.. output omitted ..             
</pre>    
<p>
    The route that Liber is using was originated by Juno. On Ceres, we can see the following:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Ceres> show route table ipvpn.inet.0 10.0.0.12

ipvpn.inet.0: 19 destinations, 33 routes (19 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.12/32       <font color='red'>*[BGP/170]</font> 5d 22:29:55, <font color='red'>localpref 150</font>, from 10.0.0.5
                      AS path: 65500 I, validation-state: unverified
                    > to 4.0.0.18 via xe-2/0/0.104, Push 16, Push 300144(top)
                    [BGP/170] 5d 22:29:55, localpref 150, from 10.0.0.4
                      AS path: 65500 I, validation-state: unverified
                    > to 4.0.0.18 via xe-2/0/0.104, Push 16, Push 300144(top)
                    [BGP/170] 1w2d 20:49:46, <font color='red'>localpref 10</font>
                      AS path: 65500 I, validation-state: unverified
                    > to <font color='red'>4.0.0.45</font> via xe-2/0/1.111    <-- Orcus IP            
</pre>      
<p>
    Ceres is also using the route via Juno. The route that it received from Orcus is not active, this is because the local preference is only 10.
    Note that there is no ‘from state inactive’ in our PE BGP policies as well. That is why the second route was is not visible on the Liber PE.
</p>
<p>
    Let’s continue and disable the BGP session between Juno and Luna:
</p>
<pre style="font-size:12px">       
play@MX104-TEST-HB:Juno# deactivate routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21          
</pre>        
<p>
    The connectivity between Saturn and Genius remains, but traffic now flows through Orcus:
</p>                
<pre style="font-size:12px">
play@MX104-TEST-HB:Saturn> ping 10.0.0.3 source 10.0.0.12
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=61 time=0.856 ms
64 bytes from 10.0.0.3: icmp_seq=1 ttl=61 time=0.703 ms
^C
--- 10.0.0.3 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.703/0.779/0.856/0.077 ms

play@MX104-TEST-HB:Saturn> traceroute 10.0.0.3 source 10.0.0.12
traceroute to 10.0.0.3 (10.0.0.3) from 10.0.0.12, 30 hops max, 40 byte packets
 1  192.168.1.3 (192.168.1.3)  0.599 ms  0.425 ms  0.370 ms
 2  <font color='red'>192.168.1.2</font> (192.168.1.2)  0.438 ms  1.550 ms  0.407 ms
 3  <font color='red'>4.0.0.46</font> (4.0.0.46)  0.429 ms  0.521 ms  0.508 ms
 4  4.0.0.98 (4.0.0.98)  0.633 ms  0.588 ms  0.551 ms
 5  10.0.0.3 (10.0.0.3)  0.718 ms  0.807 ms  0.692 ms                  
</pre>    
<p>
                    The first IP address in the traceroute is still Luna. This is because the VRRP mastership did not change. On Luna, we can see the following in our routing table:
</p>                
<pre style="font-size:12px">>                   
play@MX104-TEST-HB:Luna> show route 10.0.0.3

inet.0: 18 destinations, 19 routes (18 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.3/32        *[BGP/170] 00:03:33, <font color='red'>MED 150</font>, localpref 100
                      AS path: 1 I, validation-state: unverified
                    > to <font color='red'>192.168.1.2</font> via xe-2/0/1.150
</pre>    
<p>
    When the Luna router fails, the Orcus router will assume the Master state in VRRP. From that moment, traffic will flow directly via Orcus.  
    Alternatively, by using some tracking mechanism, you could also make Orcus pre-empt Luna’s VRRP mastership and thereby making Orcus the immediate next-hop for the customer router Saturn. Furthermore, by creating extra policy terms and referring to ‘from state inactive’, you can make sure that all the routers will constantly see all the possible routes.
</p>
<p>
    For now I’ll just leave it at this.</p>

<p>
    One more thing I wanted to point out but forgot. 
    This is that by using SOO, the Luna routes are not redistributed back to the customer location via de PE routers.
</p>
<p>
We can verify this on Orcus:
</p>                
<pre style="font-size:12px">                
play@MX104-TEST-HB:Orcus> show route receive-protocol bgp 4.0.0.46

inet.0: 19 destinations, 33 routes (19 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 4.0.0.20/30             4.0.0.46             150                1 I
  4.0.0.40/30             4.0.0.46             150                1 I
  4.0.0.44/30             4.0.0.46             150                1 I
  4.0.0.76/30             4.0.0.46             150                1 I
  4.0.0.96/30             4.0.0.46             150                1 I
  4.0.0.100/30            4.0.0.46             150                1 I
  10.0.0.2/32             4.0.0.46             150                1 I
  10.0.0.3/32             4.0.0.46             150                1 I
  10.0.0.4/32             4.0.0.46             150                1 I
  10.0.0.5/32             4.0.0.46             150                1 I
  10.0.0.7/32             4.0.0.46             150                1 I
  10.0.0.10/32            4.0.0.46             150                1 I
  10.0.0.14/32            4.0.0.46             150                1 I
  10.0.0.15/32            4.0.0.46             150                1 I
</pre>  
<p>
The complete configuration for the entire MPLS VPN lab can be found <a href="https://saidvandeklundert.net/2015-01-18-juniper-mpls-vpn-basics/" target="_blank">here</a>.
</p>
<p>
    That's all.
</p>
