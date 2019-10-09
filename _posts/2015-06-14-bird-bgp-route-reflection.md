---
layout: post
title: BIRD BGP route-reflector
tags: [bird, bgp]
image: /img/bird_logo.png
---

<p>
In this example, a server running <b>BIRD</b> will function as a <b>route-reflector</b> for two MX-routers:
</p>              
 	
![BIRD route-reflector](/img/bird-route-reflector.png "BIRD route-reflector")    

<p>
The BIRD configuration is quite easy to understand and use. 
There is only one thing about BIRD and BGP that requires additional clarification in advance.
</p>
<p>
By default, BIRD uses the 'master' table. 
In this scenario, I created a separate table called 'inet0'.
This will be the table that BIRD is going to use to store BGP routes.
What BIRD will not do is resolve the next-hop from that same table. 
By configuring 'igp table master', BIRD will resolve the next-hop by consulting the master table.
</p>		

![BIRD routing tables](/img/bird-resolve.png "BIRD routing tables")    

<p>
Also, since a BIRD RR might be better off placed outside of the forwarding path, the BIRD RR is a non-forwarding RR. 
I hope the other comments are elaborative enough;
</p>

<pre>
[root@bird ~]# more /etc/bird.bgp
# creating a separate table for BIRD to store BGP routes
table inet0;

# the following is a template. The template needs to be applied to individual neighbors.

template bgp <font color='red'>RR</font> {
        debug all;                                  # debug BGP
        description "BIRD RR";
        local as 1;                                 # the AS used by the local BGP speaker
        multihop;                                   # was not default in 1.3.11
        rr client;                                  # make defined neighbors rr clients
        igp table master;                           # necessary to resolve the next-hop
        table inet0;                                # make BGP use this table
        import all;                                 # just accept everything
        export all;                                 # and advertise it to all the neigbors
        connect retry time 10;                      # reconnect try after 10s
        hold time 30;                               # hold time send in BGP messages
        source address 172.30.0.254;                # Source BGP from this IP address
}

# neighbors can be added in the following file:
include "bird.bgp.clients";
</pre>

<p>
The route-reflector clients:
</p>

<pre>
[root@bird ~]# more /etc/bird.bgp.clients
#  from ‘RR’ refers to the template stored in bird.bgp
protocol bgp Tiberius from <font color='red'>RR</font> { neighbor 1.1.1.9 as 1; };
protocol bgp Commodus from <font color='red'>RR</font> { neighbor 1.1.1.4 as 1; };
</pre>

<p>
After activating this configuration, you can issue the following command to examine the BGP table ‘inet0’;
</p>

<pre>
[root@bird ~]# birdc show route all table inet0
BIRD 1.3.11 ready.
192.168.76.0/24    via 172.30.0.2 on eth1 [Tiberius 14:03 from 1.1.1.9] * (100/?) [i]
        Type: BGP unicast univ
        BGP.origin: IGP
        BGP.as_path:
        BGP.next_hop: 1.1.1.9
        BGP.local_pref: 100
</pre>

<p>
To verify that the route, received from Tiberius, was forwarded by BIRD:
</p>

<pre>
play@MX480-TEST-RE0:Commodus> show route receive-protocol bgp 172.30.0.254 detail

inet.0: 34 destinations, 35 routes (30 active, 0 holddown, 4 hidden)
* 192.168.76.0/24 (1 entry, 1 announced)
     Accepted
     Nexthop: 1.1.1.9
     Localpref: 100
     AS path: I (Originator)
     Cluster list:  172.30.0.254
     Originator ID: 1.1.1.9
</pre>

<p> 
A nice thing about BIRD is that it allows you to store the configuration in different files.
In this example, I made use of the following files:
</p>
      
![BIRD route reflector configuration files](/img/bird-route-reflector-configuration.png "BIRD route reflector configuration files")   

<p>
The ‘bird.conf’ file references bird.ospf and bird.bgp. The route-reflector clients are put into a separate file which is referenced by bird.bgp. 
</p>                                    
<p>
First, the content of bird.conf:
</p>
<pre>
router id 172.30.0.254;

debug protocols all;

protocol device {
        scan time 10;
}


include "bird.ospf";

include "bird.bgp";
</pre>
<p>
The OSPF configuration (same as in a previous post):
</p>
<pre>
[root@bird ~]# more /etc/bird.ospf
protocol ospf {
        debug all;
        area 0 {
                interface "eth1" {
                        passwords {
                                password "LaLa" {
                                        id 1;
                                        generate from "01-01-2000 00:00:00";
                                        accept from "01-01-2000 00:00:00";
                                };
                        };
                        cost 5;
                        type pointopoint;
                        hello 5; retransmit 2; wait 10; dead 20;
                        authentication cryptographic;
                };
                interface "lo" {
                        cost 1000;
                        stub;
                };
        };
}
</pre>
