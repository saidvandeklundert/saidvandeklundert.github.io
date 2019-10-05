---
layout: post
title: Juniper MPLS VPN basics.
image: /img/juniper_logo.jpg
---
<p>
    For a little while now, I have been wanting to do a Juniper IP VPN lab. 
    I wanted to gather most of the basics into one post. 
    In this post, I will elaborate on the different protocols and how they are configured. 
    The complete configuration is posted at the bottom of this post <a class="tocxref" href="#1">here</a>. 
</p>
<br>
<br>                


![MPLS VPN](/img/mpls-ipvpn-juniper.png "MPLS VPN") 


<p>
    <b>
        The MPLS part:
    </b>
</p>
<p>
    The interfaces that will have to handle mpls traffic will need the mpls address family configured, like this:
</p>
<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 100 family mpls
</pre>          
<p>
    Furthermore, the interface needs to be added in the protocols mpls stanza, like this;
</p>
<pre style="font-size:12px">
set protocols mpls interface xe-2/0/0.100
</pre>   
<p>
    Another configuration statement used on the PE’s is ‘<b>set protocols mpls no-propagate-ttl</b>’.  
    An ingress LSP router will normally copy the IP TTL field into the MPLS header. 
    This configuration command changes this and it will make the ingress LSP router push an MPLS header onto the received IP packets with a TTL of 255.
    This is configured on all PE’s so that CPE’s (or customers) running a traceroute will only see the PE-interface. This will make the MPLS network appears as one hop.
    Additionally, because there are P routers that have no knowledge of the ipvpn, you will not see any ‘broken’  traceroutes from a CPE.
</p>
<p>
    Example without <b>mpls no-propagate-ttl</b>:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Genius> traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 30 hops max, 40 byte packets
 1  4.0.0.98 (4.0.0.98)  0.502 ms  0.589 ms  0.376 ms
 2  * * *
 3  10.0.0.2 (10.0.0.2)  0.655 ms  0.619 ms  0.558 ms
</pre>  
<p>
    With <b>mpls no-propagate-ttl</b>:
</p>                
<pre style="font-size:12px">               
play@MX104-TEST-HB:Genius> traceroute 10.0.0.2
traceroute to 10.0.0.2 (10.0.0.2), 30 hops max, 40 byte packets
 1  4.0.0.98 (4.0.0.98)  0.507 ms  0.440 ms  0.377 ms
 2  10.0.0.2 (10.0.0.2)  0.597 ms  0.583 ms  0.541 ms
</pre>       
<p>
    I did not use the ‘no-decrement-ttl’ statement. 
    This configuration statement differs from ‘no-propagate-ttl’ even though they can both make the MPLS cloud appear as 1 hop.  
    The differences are that the  ‘no-propagate-ttl’ works for both LDP and RSVP alike and that the ‘no-decrrement-ttl’ uses a Juniper proprietary RSVP object. 
</p>
<p>
    Another possibility would have been using the ‘no-vrf-propagate-ttl’ which is configurable under the ‘routing-instance’ stanza.
    This is a more specific configuration and it will override the configuration on the global level. 
    It also possible to turn it on from this configuration stanza, using ‘set routing-instances ipvpn vrf-propagate-ttl’.
</p>
<p>
    <b>
        The LDP part:
    </b>
</p>           
<p>
    All interfaces running LDP need to be configured under the protocols ldp stanza, like this:
</p>
<pre style="font-size:12px">
set protocols ldp interface xe-2/0/0.114
</pre>                              
<p>
    This will enable LDP on the interface. The router will start sending out hello’s to 224.0.0.2 in an attempt to discover neighbors. When a neighbor is discovered, LDP will form a TCP session with that neighbor.  
    The TCP session between the  loopback  IP addresses of the routers is initiated by the router with the highest IP address. 
</p>
<p>
    This TCP session, unlike the neighbor discovery, can be authenticated using the following command:
</p>
<pre style="font-size:12px">
set protocols ldp session 10.0.0.5 authentication-key "$9$mTzntpOBIhevdwYoUD"
</pre>                  
<p>
    Note, LDP does not need to be enabled on the loopback interface since we are not using directed ldp here.
</p>
<p>
    LDP is configured to use the same metrics for routes as the IGP IS-IS. Without ‘ldp track-igp-metric’, enabled we would see the following:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Liber> show route 10.0.0.5

inet.0: 29 destinations, 29 routes (29 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.5/32        *[IS-IS/18] 1d 14:48:25,  <font color='red'>metric 10</font>
                    > to 4.0.0.58 via xe-2/0/0.114

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.5/32        *[LDP/9] 1d 14:48:25,  <font color='red'>metric 1</font>
                    > to 4.0.0.58 via xe-2/0/0.114
</pre>  
                <p>
                    With ‘ldp track-igp-metric’ enabled, we can see the following:
                </p>
<pre style="font-size:12px"> 
play@MX104-TEST-HB:Liber> show route 10.0.0.5

inet.0: 29 destinations, 29 routes (29 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.5/32        *[IS-IS/18] 1d 14:51:52, <font color='red'>metric 10</font>
                    > to 4.0.0.58 via xe-2/0/0.114

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.5/32        *[LDP/9] 00:00:31, <font color='red'>metric 10</font>
                    > to 4.0.0.58 via xe-2/0/0.114
</pre>      
<p>
     The ‘set protocols ldp igp-synchronization holddown-interval 10’ is not to be confused with the actual ldp synchronization. 
     This configuration command specifies the amount of seconds LDP waits before it informs the IGP, in this case IS-IS, that the LDP neighbor and session for an interface is up.  
     It is not needed in this network and it would be better to configure this only for larger networks.  
</p>
<p>
    <b>
        The IS-IS part:
    </b>
</p>   
<p>
    Since this lab focuses on l3vpn and since I am planning another lab on IS-IS, I will only go into this configuration command:
</p>
<pre style="font-size:12px">
set protocols isis interface xe-2/0/0.113 ldp-synchronization hold-time 10
</pre>    
<p>
    This command will have IS-IS advertise the maximum cost for a link until LDP is operational.  It is extended with a hold-time (in seconds), allowing LDP to converge.
</p>
<p>
    Before ldp sync:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Liber> show isis interface xe-2/0/0.112 extensive
IS-IS interface database:
xe-2/0/0.112
  Index: 429, State: 0x6, Circuit id: 0x1, Circuit type: 2
  LSP interval: 100 ms, CSNP interval: 15 s, Loose Hello padding
  Adjacency advertisement: Advertise
  Level 1
    Adjacencies: 0, Priority: 64, Metric: 10
    Disabled
  Level 2
    Adjacencies: 1, Priority: 64, Metric: 10
    Hello Interval: 9.000 s, Hold Time: 27 s
</pre>   
<P>
    With ldp-synchronization enabled:
</P>                               
<pre style="font-size:12px">
play@MX104-TEST-HB:Liber> show isis interface xe-2/0/0.112 extensive
IS-IS interface database:
xe-2/0/0.112
  Index: 429, State: 0x6, Circuit id: 0x1, Circuit type: 2
  LSP interval: 100 ms, CSNP interval: 15 s, Loose Hello padding
  Adjacency advertisement: Advertise
  <font color='red'>LDP sync state: in sync, for: 00:00:55, reason: LDP up during config</font>
  config holdtime: 10 seconds
  Level 1
    Adjacencies: 0, Priority: 64, Metric: 10
    Disabled
  Level 2
    Adjacencies: 1, Priority: 64, Metric: 10
    Hello Interval: 9.000 s, Hold Time: 27 s
</pre>      
<p>
    Let’s look at the metric for the route towards Libers loopback IP address from router Sol:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Sol> show route 10.0.0.7

inet.0: 26 destinations, 26 routes (26 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.7/32        *[IS-IS/18] 00:00:16, <font color='red'>metric 10</font>
                    > to 4.0.0.49 via xe-2/0/1.112

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.7/32        *[LDP/9] 00:00:16, <font color='red'>metric 10</font>
                    > to 4.0.0.49 via xe-2/0/1.112
</pre> 
<p>
    Now, let’s examine the metric right after clearing  all of Liber’s LDP sessions:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Sol> show route 10.0.0.7

inet.0: 26 destinations, 26 routes (26 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.7/32        *[IS-IS/18] 00:00:01, <font color='red'>metric 16777214</font>
                    > to 4.0.0.49 via xe-2/0/1.112

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.7/32        *[LDP/9] 00:00:00, <font color='red'>metric 16777214</font>
                    > to 4.0.0.49 via xe-2/0/1.112
</pre>   
<p>
    And after 10 seconds:
</p>                
<pre style="font-size:12px">
play@MX104-TEST-HB:Sol> show route 10.0.0.7

inet.0: 26 destinations, 26 routes (26 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.7/32        *[IS-IS/18] 00:00:01, <font color='red'>metric 10</font>
                    > to 4.0.0.49 via xe-2/0/1.112

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.7/32        *[LDP/9] 00:00:10, <font color='red'>metric 10</font>
                    > to 4.0.0.49 via xe-2/0/1.112
</pre>                                                 
<p>
    <b>
        The BGP part:
    </b>
</p>  
<p>
    Mars, Sol, Apollo and Jupiter are P routers. They are not running BGP and they can do without BGP’s routing information since they label-switch all traffic.
    Liber, Ceres, Juno and Janus are PE routers running BGP. Ceres and Janus are acting as route-reflectors and they each have their own cluster id configured.
    The inet-vpn unicast family is enabled for obvious reasons (it’s an ipvpn lab).                     
</p>
<p>
    The following is some (rather lengthy) example printout of Liber:
</p>

<pre style="font-size:12px">
play@MX104-TEST-HB:Liber> show bgp summary
Groups: 1 Peers: 2 Down peers: 0
Table          Tot Paths  Act Paths Suppressed    History Damp State    Pending
inet.0
                       0          0          0          0          0          0
bgp.l3vpn.0
                      26         13          0          0          0          0
Peer                     AS      InPkt     OutPkt    OutQ   Flaps Last Up/Dwn State|#Active/Received/Accepted/Damped...
10.0.0.2                  1         11          3       0       0  2d 3:10:47 Establ
  inet.0: 0/0/0/0
  bgp.l3vpn.0: 10/13/13/0
  ipvpn.inet.0: 10/13/13/0
10.0.0.4                  1          9          4       0       0  2d 3:10:48 Establ
  inet.0: 0/0/0/0
  bgp.l3vpn.0: 3/13/13/0
  ipvpn.inet.0: 3/13/13/0

play@MX104-TEST-HB:Liber> show bgp neighbor 10.0.0.4
Peer: 10.0.0.4+52845 AS 1      Local: 10.0.0.7+179 AS 1
  Description: Janus
  Type: Internal    State: Established    Flags: &lt;ImportEval Sync>
  Last State: OpenConfirm   Last Event: RecvKeepAlive
  Last Error: None
  Options: &lt;Preference LocalAddress HoldTime AuthKey AddressFamily PeerAS Rib-group Refresh>
  Options: &lt;BfdEnabled>
  Authentication key is configured
  Address families configured: inet-unicast inet-vpn-unicast
  Local Address: 10.0.0.7 Holdtime: 0 Preference: 170
  Number of flaps: 0
  Peer ID: 10.0.0.4        Local ID: 10.0.0.7          Active Holdtime: 0
  Keepalive Interval: 0          Group index: 0    Peer index: 0
  BFD: enabled, up
  NLRI for restart configured on peer: inet-unicast inet-vpn-unicast
  NLRI advertised by peer: inet-unicast inet-vpn-unicast
  NLRI for this session: inet-unicast inet-vpn-unicast
  Peer supports Refresh capability (2)
  Stale routes from peer are kept for: 300
  Peer does not support Restarter functionality
  NLRI that restart is negotiated for: inet-unicast inet-vpn-unicast
  NLRI of received end-of-rib markers: inet-unicast inet-vpn-unicast
  NLRI of all end-of-rib markers sent: inet-unicast inet-vpn-unicast
  Peer supports 4 byte AS extension (peer-as 1)
  Peer does not support Addpath
  Table inet.0 Bit: 10000
    RIB State: BGP restart is complete
    Send state: in sync
    Active prefixes:              0
    Received prefixes:            0
    Accepted prefixes:            0
    Suppressed due to damping:    0
    Advertised prefixes:          0
  Table bgp.l3vpn.0
    RIB State: BGP restart is complete
    RIB State: VPN restart is complete
    Send state: not advertising
    Active prefixes:              3
    Received prefixes:            13
    Accepted prefixes:            13
    Suppressed due to damping:    0
  Table ipvpn.inet.0 Bit: 30000
    RIB State: BGP restart is complete
    RIB State: VPN restart is complete
    Send state: in sync
    Active prefixes:              3
    Received prefixes:            13
    Accepted prefixes:            13
    Suppressed due to damping:    0
    Advertised prefixes:          5
  Last traffic (seconds): Received 184256 Sent 184260 Checked 184260
  Input messages:  Total 9      Updates 7       Refreshes 0     Octets 775
  Output messages: Total 4      Updates 1       Refreshes 0     Octets 307
  Output Queue[0]: 0
  Output Queue[1]: 0
  Output Queue[2]: 0

play@MX104-TEST-HB:Liber> show route advertising-protocol bgp 10.0.0.4

ipvpn.inet.0: 20 destinations, 33 routes (20 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 4.0.0.96/30             Self                         100        I
* 4.0.0.100/30            Self                         100        I
* 10.0.0.3/32             Self                         100        I
* 10.0.0.7/32             Self                         100        I
* 10.0.0.10/32            Self                         100        I

play@MX104-TEST-HB:Liber> show route receive-protocol bgp 10.0.0.4

inet.0: 29 destinations, 29 routes (29 active, 0 holddown, 0 hidden)

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)

ipvpn.inet.0: 20 destinations, 33 routes (20 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  4.0.0.20/30             10.0.0.5                     100        I
* 4.0.0.40/30             10.0.0.4                     100        I
  4.0.0.44/30             10.0.0.2                     100        I
  4.0.0.76/30             10.0.0.5                     100        I
  10.0.0.2/32             10.0.0.2                     100        I
* 10.0.0.4/32             10.0.0.4                     100        I
  10.0.0.5/32             10.0.0.5                     100        I
  10.0.0.8/32             10.0.0.5                     150        65500 I
  10.0.0.11/32            10.0.0.2                     10         65500 I
  10.0.0.12/32            10.0.0.5                     150        65500 I
* 10.0.0.14/32            10.0.0.4                     100        I
  10.0.0.15/32            10.0.0.5                     100        I
  192.168.1.0/24          10.0.0.5                     150        65500 I

iso.0: 1 destinations, 1 routes (1 active, 0 holddown, 0 hidden)

mpls.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)

bgp.l3vpn.0: 13 destinations, 26 routes (13 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
  1:1:4.0.0.20/30
                          10.0.0.5                     100        I
  1:1:4.0.0.40/30
*                         10.0.0.4                     100        I
  1:1:4.0.0.44/30
                          10.0.0.2                     100        I
  1:1:4.0.0.76/30
                          10.0.0.5                     100        I
  1:1:10.0.0.2/32
                          10.0.0.2                     100        I
  1:1:10.0.0.4/32
*                         10.0.0.4                     100        I
  1:1:10.0.0.5/32
                          10.0.0.5                     100        I
  1:1:10.0.0.8/32
                          10.0.0.5                     150        65500 I
  1:1:10.0.0.11/32
                          10.0.0.2                     10         65500 I
  1:1:10.0.0.12/32
                          10.0.0.5                     150        65500 I
  1:1:10.0.0.14/32
*                         10.0.0.4                     100        I
  1:1:10.0.0.15/32
                          10.0.0.5                     100        I
  1:1:192.168.1.0/24
                          10.0.0.5                     150        65500 I
</pre>                   
                
<p>
    All PE’s have a hold-time of 0, meaning no keepalives are exchanged. The failure detection is in the hands of bfd, which is running in hardware.
    Below is an example of the BFD session between Ceres and Juno:
</p>
<pre style="font-size:12px">               
play@MX104-TEST:Ceres> show bfd session extensive | find 10.0.0.5
10.0.0.5                 Up                       1.500     0.500        3
 Client BGP, TX interval 0.500, RX interval 0.500
 Session up time 2d 02:45
 Local diagnostic None, remote diagnostic None
 Remote state Up, version 1
 Logical system 11, routing table index 16
 Min async interval 0.500, min slow interval 1.000
 Adaptive async TX interval 0.500, RX interval 0.500
 Local min TX interval 0.500, minimum RX interval 0.500, multiplier 3
 Remote min TX interval 0.500, min RX interval 0.500, multiplier 3
 Local discriminator 132, remote discriminator 133
 Echo mode disabled/inactive
 <font color='red'>Remote is control-plane independent</font>
 Multi-hop route table 16, local-address 10.0.0.2
  Session ID: 0x580018

6 sessions, 6 clients
Cumulative transmit rate 36.0 pps, cumulative receive rate 36.0 pps
</pre>                   
                
                
<p>
<b>
The CPE part:
</b>
</p>  
<p>
    Most of the customer locations have a default route configured towards the PE. On the PE, there is a static route configured towards the CPE. This static route is redistributed via BGP, enabling all the locations to reach the prefix in use on the CPE.
    Then, last but not least, there is a multihomed location. This location is handled in a separate post <a href="juniper_ipvpn_multihomed_location.php">here</a>.
</p>              
<p>
    Anyway, Saturn was able to reach all of the CPE IP addresses at the end:
</p>
<pre style="font-size:12px">                 
play@MX104-TEST-HB:Saturn> ping 10.0.0.3
PING 10.0.0.3 (10.0.0.3): 56 data bytes
64 bytes from 10.0.0.3: icmp_seq=0 ttl=61 time=0.907 ms
^C
--- 10.0.0.3 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.907/0.907/0.907/0.000 ms

play@MX104-TEST-HB:Saturn> ping 10.0.0.10
PING 10.0.0.10 (10.0.0.10): 56 data bytes
64 bytes from 10.0.0.10: icmp_seq=0 ttl=61 time=0.749 ms
^C
--- 10.0.0.10 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.749/0.749/0.749/0.000 ms

play@MX104-TEST-HB:Saturn> ping 10.0.0.14
PING 10.0.0.14 (10.0.0.14): 56 data bytes
64 bytes from 10.0.0.14: icmp_seq=0 ttl=61 time=0.952 ms
^C
--- 10.0.0.14 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.952/0.952/0.952/0.000 ms

play@MX104-TEST-HB:Saturn> ping 10.0.0.15
PING 10.0.0.15 (10.0.0.15): 56 data bytes
64 bytes from 10.0.0.15: icmp_seq=0 ttl=62 time=0.670 ms
^C
--- 10.0.0.15 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.670/0.670/0.670/0.000 ms
</pre>  


<a name="1"></a>

<p>
    <h4>
        The PE routers:
    </h4>
</p> 
<b>
Liber:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 112 description SOL
set interfaces xe-2/0/0 unit 112 vlan-id 112
set interfaces xe-2/0/0 unit 112 family inet mtu 1500
set interfaces xe-2/0/0 unit 112 family inet address 4.0.0.49/30
set interfaces xe-2/0/0 unit 112 family iso
set interfaces xe-2/0/0 unit 112 family mpls
set interfaces xe-2/0/0 unit 113 description MARS
set interfaces xe-2/0/0 unit 113 vlan-id 113
set interfaces xe-2/0/0 unit 113 family inet mtu 1500
set interfaces xe-2/0/0 unit 113 family inet address 4.0.0.53/30
set interfaces xe-2/0/0 unit 113 family iso
set interfaces xe-2/0/0 unit 113 family mpls
set interfaces xe-2/0/0 unit 114 description JUNO
set interfaces xe-2/0/0 unit 114 vlan-id 114
set interfaces xe-2/0/0 unit 114 family inet mtu 1500
set interfaces xe-2/0/0 unit 114 family inet address 4.0.0.57/30
set interfaces xe-2/0/0 unit 114 family iso
set interfaces xe-2/0/0 unit 114 family mpls
set interfaces xe-2/0/1 unit 124 description GENIUS
set interfaces xe-2/0/1 unit 124 vlan-id 124
set interfaces xe-2/0/1 unit 124 family inet mtu 1500
set interfaces xe-2/0/1 unit 124 family inet address 4.0.0.98/30
set interfaces xe-2/0/1 unit 124 family iso
set interfaces xe-2/0/1 unit 125 description MERCURY
set interfaces xe-2/0/1 unit 125 vlan-id 125
set interfaces xe-2/0/1 unit 125 family inet mtu 1500
set interfaces xe-2/0/1 unit 125 family inet address 4.0.0.102/30
set interfaces xe-2/0/1 unit 125 family iso
set interfaces ae1 unit 300 description VESPASIAN
set interfaces ae1 unit 300 vlan-id 300
set interfaces ae1 unit 300 family inet mtu 1500
set interfaces ae1 unit 300 family inet address 5.0.0.2/30
set interfaces ae1 unit 300 family iso
set interfaces ae1 unit 301 description CONSTANTINE
set interfaces ae1 unit 301 vlan-id 301
set interfaces ae1 unit 301 family inet mtu 1500
set interfaces ae1 unit 301 family inet address 5.0.0.6/30
set interfaces ae1 unit 301 family iso
set interfaces lo0 unit 7 description LIBER
set interfaces lo0 unit 7 family inet address 10.0.0.7/32
set interfaces lo0 unit 7 family iso address 49.1984.0000.0000.0007.00
set interfaces lo0 unit 70 description ipvpn-Liber
set interfaces lo0 unit 70 family inet address 10.0.0.7/32
set protocols mpls no-propagate-ttl
set protocols mpls interface xe-2/0/0.112
set protocols mpls interface xe-2/0/0.113
set protocols mpls interface xe-2/0/0.114
set protocols mpls interface lo0.7
set protocols bgp group reflector type internal
set protocols bgp group reflector local-address 10.0.0.7
set protocols bgp group reflector hold-time 0
set protocols bgp group reflector family inet unicast
set protocols bgp group reflector family inet-vpn unicast
set protocols bgp group reflector authentication-key "$9$nObmCt0EhyreM7-oZUHPf"
set protocols bgp group reflector peer-as 1
set protocols bgp group reflector bfd-liveness-detection minimum-interval 500
set protocols bgp group reflector neighbor 10.0.0.4 description Janus
set protocols bgp group reflector neighbor 10.0.0.2 description Ceres
set protocols isis lsp-lifetime 3600
set protocols isis no-ipv6-routing
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.112 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.112 point-to-point
set protocols isis interface xe-2/0/0.112 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.112 level 1 disable
set protocols isis interface xe-2/0/0.113 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.113 point-to-point
set protocols isis interface xe-2/0/0.113 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.113 level 1 disable
set protocols isis interface xe-2/0/0.114 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.114 point-to-point
set protocols isis interface xe-2/0/0.114 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.114 level 1 disable
set protocols isis interface lo0.7
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.112
set protocols ldp interface xe-2/0/0.113
set protocols ldp interface xe-2/0/0.114
set protocols ldp session 10.0.0.5 authentication-key "$9$mTzntpOBIhevdwYoUD"
set protocols ldp session 10.0.0.9 authentication-key "$9$PQF6puB1RcKMVs2aDj"
set protocols ldp session 10.0.0.13 authentication-key "$9$Rf4SrKXx-dbYJGPTz6tp"
set protocols ldp igp-synchronization holddown-interval 10
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.124
set routing-instances ipvpn interface xe-2/0/1.125
set routing-instances ipvpn interface lo0.70
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn routing-options static route 10.0.0.10/32 next-hop 4.0.0.101
set routing-instances ipvpn routing-options static route 10.0.0.3/32 next-hop 4.0.0.97
set routing-options router-id 10.0.0.7
set routing-options autonomous-system 1
</pre>  
				
<b>				
Janus:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 107 description APOLLO
set interfaces xe-2/0/0 unit 107 vlan-id 107
set interfaces xe-2/0/0 unit 107 family inet mtu 1500
set interfaces xe-2/0/0 unit 107 family inet address 4.0.0.29/30
set interfaces xe-2/0/0 unit 107 family iso
set interfaces xe-2/0/0 unit 107 family mpls
set interfaces xe-2/0/0 unit 115 description MARS
set interfaces xe-2/0/0 unit 115 vlan-id 115
set interfaces xe-2/0/0 unit 115 family inet mtu 1500
set interfaces xe-2/0/0 unit 115 family inet address 4.0.0.61/30
set interfaces xe-2/0/0 unit 115 family iso
set interfaces xe-2/0/0 unit 115 family mpls
set interfaces xe-2/0/1 unit 100 description CERES
set interfaces xe-2/0/1 unit 100 vlan-id 100
set interfaces xe-2/0/1 unit 100 family inet mtu 1500
set interfaces xe-2/0/1 unit 100 family inet address 4.0.0.2/30
set interfaces xe-2/0/1 unit 100 family iso
set interfaces xe-2/0/1 unit 100 family mpls
set interfaces xe-2/0/1 unit 110 description VENUS
set interfaces xe-2/0/1 unit 110 vlan-id 110
set interfaces xe-2/0/1 unit 110 family inet mtu 1500
set interfaces xe-2/0/1 unit 110 family inet address 4.0.0.42/30
set interfaces xe-2/0/1 unit 110 family iso
set interfaces xe-2/0/1 unit 122 description GENIUS
set interfaces xe-2/0/1 unit 122 vlan-id 122
set interfaces xe-2/0/1 unit 122 family inet mtu 1500
set interfaces xe-2/0/1 unit 122 family inet address 4.0.0.90/30
set interfaces xe-2/0/1 unit 122 family iso
set interfaces lo0 unit 4 description JANUS
set interfaces lo0 unit 4 family inet address 10.0.0.4/32
set interfaces lo0 unit 4 family iso address 49.1984.0000.0000.0004.00
set interfaces lo0 unit 40 description ipvpn-Janus
set interfaces lo0 unit 40 family inet address 10.0.0.4/32
set protocols mpls no-propagate-ttl
set protocols mpls interface xe-2/0/0.107
set protocols mpls interface xe-2/0/0.115
set protocols mpls interface xe-2/0/1.100
set protocols mpls interface lo0.4
set protocols bgp group reflector type internal
set protocols bgp group reflector local-address 10.0.0.4
set protocols bgp group reflector hold-time 0
set protocols bgp group reflector family inet unicast
set protocols bgp group reflector family inet-vpn unicast
set protocols bgp group reflector authentication-key "$9$kP5z9CpuOIyl7db2JZ"
set protocols bgp group reflector cluster 10.0.0.4
set protocols bgp group reflector peer-as 1
set protocols bgp group reflector bfd-liveness-detection minimum-interval 500
set protocols bgp group reflector neighbor 10.0.0.7 description Liber
set protocols bgp group reflector neighbor 10.0.0.2 description Ceres
set protocols bgp group reflector neighbor 10.0.0.5 description Juno
set protocols isis lsp-lifetime 3600
set protocols isis no-ipv6-routing
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.107 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.107 point-to-point
set protocols isis interface xe-2/0/0.107 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.107 level 1 disable
set protocols isis interface xe-2/0/0.115 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.115 point-to-point
set protocols isis interface xe-2/0/0.115 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.115 level 1 disable
set protocols isis interface xe-2/0/1.100 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.100 point-to-point
set protocols isis interface xe-2/0/1.100 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.100 level 1 disable
set protocols isis interface lo0.4
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.107
set protocols ldp interface xe-2/0/0.115
set protocols ldp interface xe-2/0/1.100
set protocols ldp session 10.0.0.1 authentication-key "$9$oOGDHf5zFn90BlvWxVb"
set protocols ldp session 10.0.0.2 authentication-key "$9$tci9O1EleMWL7wYDHqfF3"
set protocols ldp session 10.0.0.9 authentication-key "$9$0Rn0IESvMLX7d24H.PQ6/"
set protocols ldp igp-synchronization holddown-interval 10
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/1.110
set routing-instances ipvpn interface lo0.40
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn routing-options static route 10.0.0.14/32 next-hop 4.0.0.41
set routing-options router-id 10.0.0.4
set routing-options autonomous-system 1				
</pre>  

<b>				
Ceres:
</b>
<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 100 description JANUS
set interfaces xe-2/0/0 unit 100 vlan-id 100
set interfaces xe-2/0/0 unit 100 family inet mtu 1500
set interfaces xe-2/0/0 unit 100 family inet address 4.0.0.1/30
set interfaces xe-2/0/0 unit 100 family iso
set interfaces xe-2/0/0 unit 100 family mpls
set interfaces xe-2/0/0 unit 104 description JUPITER
set interfaces xe-2/0/0 unit 104 vlan-id 104
set interfaces xe-2/0/0 unit 104 family inet mtu 1500
set interfaces xe-2/0/0 unit 104 family inet address 4.0.0.17/30
set interfaces xe-2/0/0 unit 104 family iso
set interfaces xe-2/0/0 unit 104 family mpls
set interfaces xe-2/0/0 unit 116 description SOL
set interfaces xe-2/0/0 unit 116 vlan-id 116
set interfaces xe-2/0/0 unit 116 family inet mtu 1500
set interfaces xe-2/0/0 unit 116 family inet address 4.0.0.65/30
set interfaces xe-2/0/0 unit 116 family iso
set interfaces xe-2/0/0 unit 116 family mpls
set interfaces xe-2/0/1 unit 111 description ORCUS
set interfaces xe-2/0/1 unit 111 vlan-id 111
set interfaces xe-2/0/1 unit 111 family inet mtu 1500
set interfaces xe-2/0/1 unit 111 family inet address 4.0.0.46/30
set interfaces xe-2/0/1 unit 111 family iso
set interfaces lo0 unit 2 description CERES
set interfaces lo0 unit 2 family inet address 10.0.0.2/32
set interfaces lo0 unit 2 family iso address 49.1984.0000.0000.0002.00
set interfaces lo0 unit 20 description ipvpn-Ceres
set interfaces lo0 unit 20 family inet address 10.0.0.2/32
set protocols mpls no-propagate-ttl
set protocols mpls interface xe-2/0/0.100
set protocols mpls interface xe-2/0/0.104
set protocols mpls interface xe-2/0/0.116
set protocols mpls interface lo0.2
set protocols bgp group reflector type internal
set protocols bgp group reflector local-address 10.0.0.2
set protocols bgp group reflector hold-time 0
set protocols bgp group reflector family inet unicast
set protocols bgp group reflector family inet-vpn unicast
set protocols bgp group reflector authentication-key "$9$.5Q3At0O1ElK-bs4GU"
set protocols bgp group reflector cluster 10.0.0.2
set protocols bgp group reflector peer-as 1
set protocols bgp group reflector bfd-liveness-detection minimum-interval 500
set protocols bgp group reflector neighbor 10.0.0.7 description Liber
set protocols bgp group reflector neighbor 10.0.0.4 description Janus
set protocols bgp group reflector neighbor 10.0.0.5 description Juno
set protocols isis lsp-lifetime 3600
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.100 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.100 point-to-point
set protocols isis interface xe-2/0/0.100 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.100 level 1 disable
set protocols isis interface xe-2/0/0.104 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.104 point-to-point
set protocols isis interface xe-2/0/0.104 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.104 level 1 disable
set protocols isis interface xe-2/0/0.116 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.116 point-to-point
set protocols isis interface xe-2/0/0.116 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.116 level 1 disable
set protocols isis interface lo0.2
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.100
set protocols ldp interface xe-2/0/0.104
set protocols ldp interface xe-2/0/0.116
set protocols ldp session 10.0.0.4 authentication-key "$9$edAWL7wsg4aGk.n9A0RE"
set protocols ldp session 10.0.0.6 authentication-key "$9$IN9cyeLXN-VsaZm5QnAt"
set protocols ldp session 10.0.0.13 authentication-key "$9$LU0NdwoaGUjk5Qt0BErl"
set protocols ldp igp-synchronization holddown-interval 10
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
set routing-options router-id 10.0.0.2
set routing-options autonomous-system 1
</pre>  
				
<b>				
Juno:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 119 description VULCAN
set interfaces xe-2/0/0 unit 119 vlan-id 119
set interfaces xe-2/0/0 unit 119 family inet mtu 1500
set interfaces xe-2/0/0 unit 119 family inet address 4.0.0.77/30
set interfaces xe-2/0/0 unit 119 family iso
set interfaces xe-2/0/1 unit 102 description JUPITER
set interfaces xe-2/0/1 unit 102 vlan-id 102
set interfaces xe-2/0/1 unit 102 family inet mtu 1500
set interfaces xe-2/0/1 unit 102 family inet address 4.0.0.10/30
set interfaces xe-2/0/1 unit 102 family iso
set interfaces xe-2/0/1 unit 102 family mpls
set interfaces xe-2/0/1 unit 103 description APOLLO
set interfaces xe-2/0/1 unit 103 vlan-id 103
set interfaces xe-2/0/1 unit 103 family inet mtu 1500
set interfaces xe-2/0/1 unit 103 family inet address 4.0.0.14/30
set interfaces xe-2/0/1 unit 103 family iso
set interfaces xe-2/0/1 unit 103 family mpls
set interfaces xe-2/0/1 unit 105 description LUNA
set interfaces xe-2/0/1 unit 105 vlan-id 105
set interfaces xe-2/0/1 unit 105 family inet mtu 1500
set interfaces xe-2/0/1 unit 105 family inet address 4.0.0.22/30
set interfaces xe-2/0/1 unit 105 family iso
set interfaces xe-2/0/1 unit 106 description SATURN
set interfaces xe-2/0/1 unit 106 vlan-id 106
set interfaces xe-2/0/1 unit 106 family inet mtu 1500
set interfaces xe-2/0/1 unit 106 family inet address 4.0.0.26/30
set interfaces xe-2/0/1 unit 106 family iso
set interfaces xe-2/0/1 unit 114 description LIBER
set interfaces xe-2/0/1 unit 114 vlan-id 114
set interfaces xe-2/0/1 unit 114 family inet mtu 1500
set interfaces xe-2/0/1 unit 114 family inet address 4.0.0.58/30
set interfaces xe-2/0/1 unit 114 family iso
set interfaces xe-2/0/1 unit 114 family mpls
set interfaces lo0 unit 5 description JUNO
set interfaces lo0 unit 5 family inet address 10.0.0.5/32
set interfaces lo0 unit 5 family iso address 49.1984.0000.0000.0005.00
set interfaces lo0 unit 50 description ipvpn-Juno
set interfaces lo0 unit 50 family inet address 10.0.0.5/32
set protocols mpls no-propagate-ttl
set protocols mpls interface xe-2/0/1.102
set protocols mpls interface xe-2/0/1.103
set protocols mpls interface xe-2/0/1.114
set protocols mpls interface lo0.5
set protocols bgp group reflector type internal
set protocols bgp group reflector local-address 10.0.0.5
set protocols bgp group reflector hold-time 0
set protocols bgp group reflector family inet unicast
set protocols bgp group reflector family inet-vpn unicast
set protocols bgp group reflector authentication-key "$9$2RaZD.m5TzntuSlK8N-"
set protocols bgp group reflector peer-as 1
set protocols bgp group reflector bfd-liveness-detection minimum-interval 500
set protocols bgp group reflector neighbor 10.0.0.4 description Janus
set protocols bgp group reflector neighbor 10.0.0.2 description Ceres
set protocols isis lsp-lifetime 3600
set protocols isis no-ipv6-routing
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/1.102 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.102 point-to-point
set protocols isis interface xe-2/0/1.102 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.102 level 1 disable
set protocols isis interface xe-2/0/1.103 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.103 point-to-point
set protocols isis interface xe-2/0/1.103 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.103 level 1 disable
set protocols isis interface xe-2/0/1.114 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.114 point-to-point
set protocols isis interface xe-2/0/1.114 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.114 level 1 disable
set protocols isis interface lo0.5
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/1.102
set protocols ldp interface xe-2/0/1.103
set protocols ldp interface xe-2/0/1.114
set protocols ldp session 10.0.0.1 authentication-key "$9$VXYgajiq.PT69IhSe8L"
set protocols ldp session 10.0.0.6 authentication-key "$9$C-41uOIyrKvWXVwGjHmQz"
set protocols ldp session 10.0.0.7 authentication-key "$9$LxONdwoaGUjk5Qt0BErl"
set protocols ldp igp-synchronization holddown-interval 10
set policy-options policy-statement multihomed-ipvpn term no-cpe-originated from community cpe-originated-route
set policy-options policy-statement multihomed-ipvpn term no-cpe-originated then reject
set policy-options policy-statement multihomed-ipvpn term accept-rest then accept
set policy-options policy-statement multihomed-ipvpn-primary then local-preference 150
set policy-options policy-statement multihomed-ipvpn-primary then accept
set policy-options community cpe-originated-route members origin:1000:1000
set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface xe-2/0/0.119
set routing-instances ipvpn interface xe-2/0/1.105
set routing-instances ipvpn interface lo0.50
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn routing-options static route 10.0.0.15/32 next-hop 4.0.0.78
set routing-instances ipvpn protocols bgp group CPE type external
set routing-instances ipvpn protocols bgp group CPE passive
set routing-instances ipvpn protocols bgp group CPE authentication-key "$9$tAvZO1EleMWL7wYDHqfF3"
set routing-instances ipvpn protocols bgp group CPE peer-as 65500
set routing-instances ipvpn protocols bgp group CPE as-override
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21 metric-out 10
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21 import multihomed-ipvpn-primary
set routing-instances ipvpn protocols bgp group CPE neighbor 4.0.0.21 export multihomed-ipvpn
set routing-options router-id 10.0.0.5
set routing-options autonomous-system 1
</pre>  


<p>
    <h4>
        The P routers:
    </h4>
</p>    

<b>
Mars:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 126 description APOLLO
set interfaces xe-2/0/0 unit 126 vlan-id 126
set interfaces xe-2/0/0 unit 126 family inet mtu 1500
set interfaces xe-2/0/0 unit 126 family inet address 4.0.0.105/30
set interfaces xe-2/0/0 unit 126 family iso
set interfaces xe-2/0/0 unit 126 family mpls
set interfaces xe-2/0/0 unit 128 description SOL
set interfaces xe-2/0/0 unit 128 vlan-id 128
set interfaces xe-2/0/0 unit 128 family inet mtu 1500
set interfaces xe-2/0/0 unit 128 family inet address 4.0.0.113/30
set interfaces xe-2/0/0 unit 128 family iso
set interfaces xe-2/0/0 unit 128 family mpls
set interfaces xe-2/0/1 unit 113 description LIBER
set interfaces xe-2/0/1 unit 113 vlan-id 113
set interfaces xe-2/0/1 unit 113 family inet mtu 1500
set interfaces xe-2/0/1 unit 113 family inet address 4.0.0.54/30
set interfaces xe-2/0/1 unit 113 family iso
set interfaces xe-2/0/1 unit 113 family mpls
set interfaces xe-2/0/1 unit 115 description JANUS
set interfaces xe-2/0/1 unit 115 vlan-id 115
set interfaces xe-2/0/1 unit 115 family inet mtu 1500
set interfaces xe-2/0/1 unit 115 family inet address 4.0.0.62/30
set interfaces xe-2/0/1 unit 115 family iso
set interfaces xe-2/0/1 unit 115 family mpls
set interfaces lo0 unit 9 description MARS
set interfaces lo0 unit 9 family inet address 10.0.0.9/32
set interfaces lo0 unit 9 family iso address 49.1984.0000.0000.0009.00
set protocols mpls interface xe-2/0/0.128
set protocols mpls interface xe-2/0/1.113
set protocols mpls interface xe-2/0/1.115
set protocols mpls interface lo0.9
set protocols mpls interface xe-2/0/0.126
set protocols isis lsp-lifetime 3600
set protocols isis no-ipv6-routing
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.126 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.126 point-to-point
set protocols isis interface xe-2/0/0.126 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.126 level 1 disable
set protocols isis interface xe-2/0/0.128 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.128 point-to-point
set protocols isis interface xe-2/0/0.128 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.128 level 1 disable
set protocols isis interface xe-2/0/1.113 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.113 point-to-point
set protocols isis interface xe-2/0/1.113 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.113 level 1 disable
set protocols isis interface xe-2/0/1.115 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.115 point-to-point
set protocols isis interface xe-2/0/1.115 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.115 level 1 disable
set protocols isis interface lo0.9
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.126
set protocols ldp interface xe-2/0/0.128
set protocols ldp interface xe-2/0/1.113
set protocols ldp interface xe-2/0/1.115
set protocols ldp session 10.0.0.1 authentication-key "$9$lJlM8xbw2goZHq3/CuIR"
set protocols ldp session 10.0.0.4 authentication-key "$9$cdplKWN-bwY4UjTFnC0O"
set protocols ldp session 10.0.0.7 authentication-key "$9$Ddk.f3n9Ct0Ec8xNbg4"
set protocols ldp session 10.0.0.13 authentication-key "$9$8.F7-b4oZGDHfTAuORyr"
set protocols ldp igp-synchronization holddown-interval 10
</pre>  
				
<b>				
Sol:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 129 description JUPITER
set interfaces xe-2/0/0 unit 129 vlan-id 129
set interfaces xe-2/0/0 unit 129 family inet mtu 1500
set interfaces xe-2/0/0 unit 129 family inet address 4.0.0.221/30
set interfaces xe-2/0/0 unit 129 family iso
set interfaces xe-2/0/0 unit 129 family mpls
set interfaces xe-2/0/1 unit 112 description LIBER
set interfaces xe-2/0/1 unit 112 vlan-id 112
set interfaces xe-2/0/1 unit 112 family inet mtu 1500
set interfaces xe-2/0/1 unit 112 family inet address 4.0.0.50/30
set interfaces xe-2/0/1 unit 112 family iso
set interfaces xe-2/0/1 unit 112 family mpls
set interfaces xe-2/0/1 unit 116 description CERES
set interfaces xe-2/0/1 unit 116 vlan-id 116
set interfaces xe-2/0/1 unit 116 family inet mtu 1500
set interfaces xe-2/0/1 unit 116 family inet address 4.0.0.66/30
set interfaces xe-2/0/1 unit 116 family iso
set interfaces xe-2/0/1 unit 116 family mpls
set interfaces xe-2/0/1 unit 128 description MARS
set interfaces xe-2/0/1 unit 128 vlan-id 128
set interfaces xe-2/0/1 unit 128 family inet mtu 1500
set interfaces xe-2/0/1 unit 128 family inet address 4.0.0.114/30
set interfaces xe-2/0/1 unit 128 family iso
set interfaces xe-2/0/1 unit 128 family mpls
set interfaces lo0 unit 13 description SOL
set interfaces lo0 unit 13 family inet address 10.0.0.13/32
set interfaces lo0 unit 13 family iso address 49.1984.0000.0000.0013.00
set protocols mpls interface xe-2/0/0.129
set protocols mpls interface xe-2/0/1.112
set protocols mpls interface xe-2/0/1.116
set protocols mpls interface xe-2/0/1.128
set protocols mpls interface lo0.13
set protocols isis lsp-lifetime 3600
set protocols isis no-ipv6-routing
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.129 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.129 point-to-point
set protocols isis interface xe-2/0/0.129 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.129 level 1 disable
set protocols isis interface xe-2/0/1.112 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.112 point-to-point
set protocols isis interface xe-2/0/1.112 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.112 level 1 disable
set protocols isis interface xe-2/0/1.116 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.116 point-to-point
set protocols isis interface xe-2/0/1.116 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.116 level 1 disable
set protocols isis interface xe-2/0/1.128 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.128 point-to-point
set protocols isis interface xe-2/0/1.128 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.128 level 1 disable
set protocols isis interface lo0.13
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.129
set protocols ldp interface xe-2/0/1.112
set protocols ldp interface xe-2/0/1.116
set protocols ldp interface xe-2/0/1.128
set protocols ldp session 10.0.0.2 authentication-key "$9$UrHqPF3/9AuRhWX7V2g"
set protocols ldp session 10.0.0.6 authentication-key "$9$oCGDHf5zFn90BlvWxVb"
set protocols ldp session 10.0.0.7 authentication-key "$9$8x37-b4oZGDHfTAuORyr"
set protocols ldp session 10.0.0.9 authentication-key "$9$HmfQ/9tp01Srx-VYaJ"
set protocols ldp igp-synchronization holddown-interval 10
</pre>  
				
<b>				
Apollo:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 101 description JUPITER
set interfaces xe-2/0/0 unit 101 vlan-id 101
set interfaces xe-2/0/0 unit 101 family inet mtu 1500
set interfaces xe-2/0/0 unit 101 family inet address 4.0.0.5/30
set interfaces xe-2/0/0 unit 101 family iso
set interfaces xe-2/0/0 unit 101 family mpls
set interfaces xe-2/0/0 unit 103 description JUNO
set interfaces xe-2/0/0 unit 103 vlan-id 103
set interfaces xe-2/0/0 unit 103 family inet mtu 1500
set interfaces xe-2/0/0 unit 103 family inet address 4.0.0.13/30
set interfaces xe-2/0/0 unit 103 family iso
set interfaces xe-2/0/0 unit 103 family mpls
set interfaces xe-2/0/1 unit 107 description JANUS
set interfaces xe-2/0/1 unit 107 vlan-id 107
set interfaces xe-2/0/1 unit 107 family inet mtu 1500
set interfaces xe-2/0/1 unit 107 family inet address 4.0.0.30/30
set interfaces xe-2/0/1 unit 107 family iso
set interfaces xe-2/0/1 unit 107 family mpls
set interfaces xe-2/0/1 unit 126 description MARS
set interfaces xe-2/0/1 unit 126 vlan-id 126
set interfaces xe-2/0/1 unit 126 family inet mtu 1500
set interfaces xe-2/0/1 unit 126 family inet address 4.0.0.106/30
set interfaces xe-2/0/1 unit 126 family iso
set interfaces xe-2/0/1 unit 126 family mpls
set interfaces lo0 unit 1 description APOLLO
set interfaces lo0 unit 1 family inet address 10.0.0.1/32
set interfaces lo0 unit 1 family iso address 49.1984.0000.0000.0001.00
set protocols mpls interface xe-2/0/0.101
set protocols mpls interface xe-2/0/0.103
set protocols mpls interface xe-2/0/1.107
set protocols mpls interface lo0.1
set protocols mpls interface xe-2/0/1.126
set protocols isis lsp-lifetime 3600
set protocols isis no-ipv6-routing
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.101 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.101 point-to-point
set protocols isis interface xe-2/0/0.101 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.101 level 1 disable
set protocols isis interface xe-2/0/0.103 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.103 point-to-point
set protocols isis interface xe-2/0/0.103 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.103 level 1 disable
set protocols isis interface xe-2/0/1.107 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.107 point-to-point
set protocols isis interface xe-2/0/1.107 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.107 level 1 disable
set protocols isis interface xe-2/0/1.126 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.126 point-to-point
set protocols isis interface xe-2/0/1.126 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.126 level 1 disable
set protocols isis interface lo0.1
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.101
set protocols ldp interface xe-2/0/0.103
set protocols ldp interface xe-2/0/1.107
set protocols ldp interface xe-2/0/1.126
set protocols ldp session 10.0.0.4 authentication-key "$9$6C3nApOhcrlKWNdaGDkf5"
set protocols ldp session 10.0.0.5 authentication-key "$9$kP5z9CpuOIyl7db2JZ"
set protocols ldp session 10.0.0.6 authentication-key "$9$cudlKWN-bwY4UjTFnC0O"
set protocols ldp session 10.0.0.9 authentication-key "$9$-4wY4UDHk.f36BRhrMW"
set protocols ldp igp-synchronization holddown-interval 10
</pre>  
				
<b>				
Jupiter:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 102 description JUNO
set interfaces xe-2/0/0 unit 102 vlan-id 102
set interfaces xe-2/0/0 unit 102 family inet mtu 1500
set interfaces xe-2/0/0 unit 102 family inet address 4.0.0.9/30
set interfaces xe-2/0/0 unit 102 family iso
set interfaces xe-2/0/0 unit 102 family mpls
set interfaces xe-2/0/1 unit 101 description APOLLO
set interfaces xe-2/0/1 unit 101 vlan-id 101
set interfaces xe-2/0/1 unit 101 family inet mtu 1500
set interfaces xe-2/0/1 unit 101 family inet address 4.0.0.6/30
set interfaces xe-2/0/1 unit 101 family iso
set interfaces xe-2/0/1 unit 101 family mpls
set interfaces xe-2/0/1 unit 104 description CERES
set interfaces xe-2/0/1 unit 104 vlan-id 104
set interfaces xe-2/0/1 unit 104 family inet mtu 1500
set interfaces xe-2/0/1 unit 104 family inet address 4.0.0.18/30
set interfaces xe-2/0/1 unit 104 family iso
set interfaces xe-2/0/1 unit 104 family mpls
set interfaces xe-2/0/1 unit 129 description SOL
set interfaces xe-2/0/1 unit 129 vlan-id 129
set interfaces xe-2/0/1 unit 129 family inet mtu 1500
set interfaces xe-2/0/1 unit 129 family inet address 4.0.0.222/30
set interfaces xe-2/0/1 unit 129 family iso
set interfaces xe-2/0/1 unit 129 family mpls
set interfaces lo0 unit 6 description JUPITER
set interfaces lo0 unit 6 family inet address 10.0.0.6/32
set interfaces lo0 unit 6 family iso address 49.1984.0000.0000.0006.00
set protocols mpls interface xe-2/0/0.102
set protocols mpls interface xe-2/0/1.101
set protocols mpls interface xe-2/0/1.104
set protocols mpls interface xe-2/0/1.129
set protocols mpls interface lo0.6
set protocols isis lsp-lifetime 3600
set protocols isis spf-options delay 50
set protocols isis overload timeout 300
set protocols isis level 2 authentication-key "$9$UkHqPF3/9AuRhWX7V2g"
set protocols isis level 2 authentication-type md5
set protocols isis level 2 wide-metrics-only
set protocols isis interface xe-2/0/0.102 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/0.102 point-to-point
set protocols isis interface xe-2/0/0.102 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/0.102 level 1 disable
set protocols isis interface xe-2/0/1.101 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.101 point-to-point
set protocols isis interface xe-2/0/1.101 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.101 level 1 disable
set protocols isis interface xe-2/0/1.104 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.104 point-to-point
set protocols isis interface xe-2/0/1.104 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.104 level 1 disable
set protocols isis interface xe-2/0/1.129 ldp-synchronization hold-time 10
set protocols isis interface xe-2/0/1.129 point-to-point
set protocols isis interface xe-2/0/1.129 bfd-liveness-detection minimum-interval 100
set protocols isis interface xe-2/0/1.129 level 1 disable
set protocols isis interface lo0.6
set protocols ldp track-igp-metric
set protocols ldp interface xe-2/0/0.102
set protocols ldp interface xe-2/0/1.101
set protocols ldp interface xe-2/0/1.104
set protocols ldp interface xe-2/0/1.129
set protocols ldp session 10.0.0.1 authentication-key "$9$0c3zIESvMLX7d24H.PQ6/"
set protocols ldp session 10.0.0.2 authentication-key "$9$/NvstuBcSlev8-VJUjq5T"
set protocols ldp session 10.0.0.5 authentication-key "$9$eCAWL7wsg4aGk.n9A0RE"
set protocols ldp session 10.0.0.13 authentication-key "$9$.5Q3At0O1ElK-bs4GU"
set protocols ldp igp-synchronization holddown-interval 10
</pre>  
				
				
<p>
    <h4>
        The cpe configurations:
    </h4>
</p>    
<b>
Genius:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 122 description MARS
set interfaces xe-2/0/0 unit 122 vlan-id 122
set interfaces xe-2/0/0 unit 122 family inet mtu 1500
set interfaces xe-2/0/0 unit 122 family inet address 4.0.0.89/30
set interfaces xe-2/0/0 unit 122 family iso
set interfaces xe-2/0/0 unit 124 description LIBER
set interfaces xe-2/0/0 unit 124 vlan-id 124
set interfaces xe-2/0/0 unit 124 family inet mtu 1500
set interfaces xe-2/0/0 unit 124 family inet address 4.0.0.97/30
set interfaces xe-2/0/0 unit 124 family iso
set interfaces ae1 unit 303 description TIBERIUS
set interfaces ae1 unit 303 vlan-id 303
set interfaces ae1 unit 303 family inet mtu 1500
set interfaces ae1 unit 303 family inet address 5.0.0.14/30
set interfaces ae1 unit 303 family iso
set interfaces lo0 unit 3 description GENIUS
set interfaces lo0 unit 3 family inet address 10.0.0.3/32
set routing-options static route 0.0.0.0/0 next-hop 4.0.0.98
</pre>  
				
<b>				
Mercury:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 125 description LIBER
set interfaces xe-2/0/0 unit 125 vlan-id 125
set interfaces xe-2/0/0 unit 125 family inet mtu 1500
set interfaces xe-2/0/0 unit 125 family inet address 4.0.0.101/30
set interfaces xe-2/0/0 unit 125 family iso
set interfaces lo0 unit 10 description MERCURY
set interfaces lo0 unit 10 family inet address 10.0.0.10/32
set routing-options static route 0.0.0.0/0 next-hop 4.0.0.102
</pre>  
				
<b>				
Venus:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 110 description JANUS
set interfaces xe-2/0/0 unit 110 vlan-id 110
set interfaces xe-2/0/0 unit 110 family inet mtu 1500
set interfaces xe-2/0/0 unit 110 family inet address 4.0.0.41/30
set interfaces xe-2/0/0 unit 110 family iso
set interfaces lo0 unit 14 description VENUS
set interfaces lo0 unit 14 family inet address 10.0.0.14/32
set routing-options static route 0.0.0.0/0 next-hop 4.0.0.42
</pre>  
				
<b>				
Vulcan:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/1 unit 108 description SATURN
set interfaces xe-2/0/1 unit 108 vlan-id 108
set interfaces xe-2/0/1 unit 108 family inet mtu 1500
set interfaces xe-2/0/1 unit 108 family inet address 4.0.0.34/30
set interfaces xe-2/0/1 unit 108 family iso
set interfaces xe-2/0/1 unit 119 description JUNO
set interfaces xe-2/0/1 unit 119 vlan-id 119
set interfaces xe-2/0/1 unit 119 family inet mtu 1500
set interfaces xe-2/0/1 unit 119 family inet address 4.0.0.78/30
set interfaces xe-2/0/1 unit 119 family iso
set interfaces lo0 unit 15 description VULCAN
set interfaces lo0 unit 15 family inet address 10.0.0.15/32
set routing-options static route 0.0.0.0/0 next-hop 4.0.0.77
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
				
<b>				
Luna:
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
set routing-options autonomous-system 65500
</pre>  

<b>			
Saturn:
</b>

<pre style="font-size:12px">
set interfaces xe-2/0/0 unit 106 description JUNO
set interfaces xe-2/0/0 unit 106 vlan-id 106
set interfaces xe-2/0/0 unit 106 family inet mtu 1500
set interfaces xe-2/0/0 unit 106 family inet address 4.0.0.25/30
set interfaces xe-2/0/0 unit 106 family iso
set interfaces xe-2/0/0 unit 108 description VULCAN
set interfaces xe-2/0/0 unit 108 vlan-id 108
set interfaces xe-2/0/0 unit 108 family inet mtu 1500
set interfaces xe-2/0/0 unit 108 family inet address 4.0.0.33/30
set interfaces xe-2/0/0 unit 108 family iso
set interfaces ae1 unit 150 description Luna_Orcus
set interfaces ae1 unit 150 vlan-id 150
set interfaces ae1 unit 150 family inet mtu 1500
set interfaces ae1 unit 150 family inet address 192.168.1.254/24
set interfaces lo0 unit 12 description SATURN
set interfaces lo0 unit 12 family inet address 10.0.0.12/32
set routing-options static route 0.0.0.0/0 next-hop 192.168.1.1
</pre>  