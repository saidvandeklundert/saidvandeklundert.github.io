---
layout: post
title: Juniper LDP, follow that label
tags: [juniper, mpls]
image: /img/juniper_logo.jpg
---

<p>
    This article explains how you can analyze the forwarding table on Junos.
    On <a href="https://saidvandeklundert.net/2015-01-18-juniper-mpls-vpn-basics/">this</a> lab, I altered several metrics to make the traffic flow look like this:                    
</p>

![MPLS LDP label](/img/follow_ldp_label_1.png "MPLS LDP label") 

<p>
    The traffic flow I will be looking at is the traffic between Venus and Vulcan. 
    Both CPE’s are placed inside an MPLS VPN named ‘ipvpn’. Janus and Juno are the PE routers. 
    They are the ingress LSR’s and they will push two labels onto the packets received by the CPE’s. 
    In between Juno and Janus are two P routers (as far as this scenario is concerned) named Mars and Liber. 
    The P routers only look at the first label. Mars will perform a swap operation and Liber will perform penultimate hop-popping, leaving Juno only with the VPN label.
</p>
<p>
    IP connectivity between the Venus CPE and the Vulcan CPE is already established;
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Venus> traceroute 10.0.0.15 source 10.0.0.14
traceroute to 10.0.0.15 (10.0.0.15) from 10.0.0.14, 30 hops max, 40 byte packets
 1  4.0.0.42 (4.0.0.42)  0.576 ms  0.433 ms  0.382 ms
 2  4.0.0.77 (4.0.0.77)  0.487 ms  0.500 ms  0.465 ms
 3  10.0.0.15 (10.0.0.15)  0.668 ms  0.641 ms  0.611 ms

play@MX104-TEST-HB:Venus>
</pre>   
<p>
    To the CPE’s, the MPLS network is only one hop. So, how can we see what labels are involved for traffic flowing from Venus to Vulcan?
</p>
<p>
    Let’s start at Janus. The Janus and Juno PE’s are exchanging VPN labels using BGP. Finding out what labels are used on Janus  can be done in a lot of ways. Let’s start by looking at the following command:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Janus> show route table ipvpn.inet.0 10.0.0.15 active-path detail

ipvpn.inet.0: 19 destinations, 31 routes (19 active, 0 holddown, 0 hidden)
10.0.0.15/32 (2 entries, 1 announced)
        *BGP    Preference: 170/-101
                Route Distinguisher: 1:1
                Next hop type: Indirect
                Address: 0x283d48c
                Next-hop reference count: 35
                Source: 10.0.0.5
                Next hop type: Router, Next hop index: 2344
                Next hop: 4.0.0.62 via xe-2/0/0.115, selected
                Label operation: Push <font color='red'>16</font>, Push <font color='red'>300048</font>(top)
                Label TTL action: no-prop-ttl, no-prop-ttl(top)
                Session Id: 0x100001
                Protocol next hop: 10.0.0.5
                Push 16
                Indirect next hop: 0x27ec514 1048581 INH Session ID: 0x100017
                State: &gt;Secondary Active Int Ext ProtectionCand>
                Local AS:     1 Peer AS:     1
                Age: 10:01:56   Metric2: 30
                Validation State: unverified
                Task: BGP_1.10.0.0.5+59727
                Announcement bits (1): 0-KRT
                AS path: I
                Communities: target:1:1
                Import Accepted
                <font color='red'>VPN Label: 16</font>
                Localpref: 100
                Router ID: 10.0.0.5
                Primary Routing Table bgp.l3vpn.0                    
</pre>             
<p>
                    Another place where we could look is the forwarding table entry for the VPN route:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Janus> show route forwarding-table vpn ipvpn matching 10.0.0.15
Logical system: Janus
Routing table: ipvpn.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
10.0.0.15/32       user     0                    indr  1048581     8
                              4.0.0.62          Push <font color='red'>16</font>, Push <font color='red'>300048</font>(top)     2344     2 xe-2/0/0.115                    
</pre>                       
<p>
    The IS-IS routing table entry for the route towards the PE that originated the VPN route:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Janus> show route protocol isis 10.0.0.5

inet.0: 27 destinations, 27 routes (27 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.5/32        *[IS-IS/18] 00:01:28, metric 30
                    > to 4.0.0.62 via xe-2/0/0.115                    
</pre>  
<p>
    The LDP routing table entry for the route towards the PE that originated the VPN route:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Janus> show route protocol ldp 10.0.0.5

inet.0: 27 destinations, 27 routes (27 active, 0 holddown, 0 hidden)

inet.3: 7 destinations, 7 routes (7 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

10.0.0.5/32        *[LDP/9] 00:01:47, metric 30
                    > to 4.0.0.62 via xe-2/0/0.115, Push 300048

ipvpn.inet.0: 19 destinations, 31 routes (19 active, 0 holddown, 0 hidden)                    
</pre>                  
<p>
    The VPN label that was learned from the other PE:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Janus> show route receive-protocol bgp 10.0.0.5 table ipvpn.inet.0 10.0.0.15 extensive

ipvpn.inet.0: 19 destinations, 31 routes (19 active, 0 holddown, 0 hidden)
* 10.0.0.15/32 (2 entries, 1 announced)
     Import Accepted
     Route Distinguisher: 1:1
     <font color='red'>VPN Label: 16</font>
     Nexthop: 10.0.0.5
     Localpref: 100
     AS path: I
     Communities: target:1:1                    
</pre>    
<p>
    Filling in this first step will give us the following:
</p>

![MPLS LDP label](/img/follow_ldp_label_2.png "MPLS LDP label") 

<p>
    Now onto Mars. We already obtained the first label values on Janus. 
    We can now use the value of the MPLS label on Mars. We can use it to query the mpls.0 table or the forwarding table:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Mars> show route table mpls.0 label <font color='red'>300048</font>

mpls.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

<font color='red'>300048</font>             *[LDP/9] 18:13:33, metric 1
                    > to 4.0.0.53 via xe-2/0/1.113, <font color='red'>Swap 300848</font>

play@MX104-TEST-HB:Mars> show route forwarding-table label 300048
Logical system: Mars
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
<font color='red'>300048</font>             user     0 4.0.0.53          <font color='red'>Swap 300848</font>     2350     2 xe-2/0/1.113                    
</pre>   
<p>
    We can see that Mars’ only job is to swap labels. Mars only examines and swaps the outer label.
</p>

![MPLS LDP label](/img/follow_ldp_label_3.png "MPLS LDP label") 

<p>
    The next stop is Liber. Here we can use the label value that Mars has set:
</p>                   
<pre style="font-size:12px">
play@MX104-TEST-HB:Liber> show route table mpls.0 label 300848

mpls.0: 15 destinations, 15 routes (15 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

300848             *[LDP/9] 1d 02:31:58, metric 1
                    > to 4.0.0.58 via xe-2/0/0.114, Pop
300848(S=0)        *[LDP/9] 1d 02:31:58, metric 1
                    > to 4.0.0.58 via xe-2/0/0.114, Pop

play@MX104-TEST-HB:Liber> show route forwarding-table label 300848
Logical system: Liber
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
300848             user     0 4.0.0.58          Pop       2259     2 xe-2/0/0.114
300848(S=0)        user     0 4.0.0.58          Pop       2260     2 xe-2/0/0.114

Logical system: Liber
Routing table: __mpls-oam__.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
default            perm     0                    dscd     1907     1                    
</pre>   
<p>
    Liber is the penultimate router. 
    This means it is the last router before the LSP’s egress router. 
    This router is performing penultimate hop popping. 
    This means Liber simply pops the label and forwards the remainder of the packet to Juno. 
    The remainder of the packet will still have the VPN label:
</p>

![MPLS LDP label](/img/follow_ldp_label_4.png "MPLS LDP label") 

<p>
    The Juno router, that originally used BGP to signal the remaining label to Janus, now uses this label to correlate the received traffic with the correct VPN. 
    In this case the mpls.0 table tells us the following:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Juno> show route table mpls.0 label 16

mpls.0: 13 destinations, 13 routes (13 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

16                 *[VPN/0] 4d 19:52:35
                      to table ipvpn.inet.0, <font color='red'>Pop</font>                    
</pre>    
<p>
    When Juno forwards traffic towards 10.0.0.15, it will do so without any labels:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB:Juno> show route forwarding-table vpn ipvpn matching 10.0.0.15
Logical system: Juno
Routing table: ipvpn.inet
Internet:
Destination        Type RtRef Next hop           Type Index    NhRef Netif
10.0.0.15/32       user     0 4.0.0.78           ucst     2426     3 xe-2/0/0.119                    
</pre>                    
                
            
