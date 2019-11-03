---
layout: post
title: Primary and secondary LSPs for RSVP signaled LSPs
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---
   
<p>
A failure somewhere in the network can cause for traffic traversing an RSVP-signaled LSP to drop. 
Several possibilities exist to reduce the impact a failure can have on RSVP-signaled LSPs. 
This article is about the creation of a secondary standby path in order to reduce downtime that is incurred upon a network failure somewhere along the RSVP-signaled LSP.
</p>      
<p>
Let's make the label-switched-path between Tiberius and Commodus more robust by configuring the Tiberius router to setup two paths towards the Commodus router:
</p>

![RSVP primary and secondary LSPs](/img/rsvp-secondary-lsp-1.png "RSVP primary and secondary LSPs") 

<p>
By default, when an RSVP-signaled LSP is configured, the LSP can make use of all the links that are enabled for MPLS and RSVP. When the ingress LSR starts to signal the LSP, it will create only one LSP. This LSP is signaled along the path that follows the IGP (in this case IS-IS).
</p>

![RSVP primary and secondary LSPs](/img/rsvp-secondary-lsp-2.png "RSVP primary and secondary LSPs") 

<p>
Whenever a link that is a part of the LSP fails, packets that are making use of the LSP will start dropping. For instance, suppose the link between Caligula and Commodus fails. 
</p>
               
![RSVP primary and secondary LSPs](/img/rsvp-secondary-lsp-3.png "RSVP primary and secondary LSPs") 

 
<p>
Caligula will send a ResvTear message towards the ingress router of the LSP, Tiberius. All the routers in the path of the LSP will pass this ResvTear message to the next upstream router. As soon as the Tiberius router receives this message, it will start to consider the LSP as being down. Other protocols that were making use of the next-hop offered by this LSP will need to recalculate to find another route. If there isn’t one, traffic stops.
</p>
<p>
The Tiberius router will, following the deletion of the LSP, start to signal a new LSP towards Commodus. 
Only after finishing this new LSP will the next-hop be available again. After this, traffic will be able to flow across the LSP again.
</p>
<p>
Part of this downtime can be reduced by having a secondary LSP active before the primary LSP goes down. 
This way, as soon as Tiberius receives the ResvTear message, it will immediately start making use of the secondary LSP.
</p>   
<p>
To enable a secondary LSP, two different paths need to be configured.  This can be done using the following configuration commands:
</p>
<pre style="font-size:12px">
set protocols mpls path via_Romulus 1.1.1.6 strict
set protocols mpls path via_Hadrian 1.1.1.10 strict                
</pre>
<p>
The Tiberius router has two uplinks to the MPLS network.
By adding a different strict hop to both paths, each path will make use of a different uplink.
After passing the mandatory first hop in the LSP, the router is free to choose any next-hop from that point on. 
</p>
<p>
The creation of these paths will have no effect on traffic if they are not applied to an LSP. There is already an LSP defined on this network, this ‘ldp-tunneling’ enabled LSP is providing for a layer2 circuit between Sol and Mars. This LSP is configured as follows:
</p>
<pre style="font-size:12px">
set protocols mpls label-switched-path L2circuit to 1.1.1.4
set protocols mpls label-switched-path L2circuit ldp-tunneling                    
</pre>
<p>
To make the Tiberius router signal two LSPs towards the Commodus router, we simply add the two paths we configured to this configuration, like this;
</p>
<pre style="font-size:12px">
set protocols mpls label-switched-path L2circuit primary via_Hadrian
set protocols mpls label-switched-path L2circuit secondary via_Romulus <font color='red'>standby</font>				
</pre>
<p>
By configuring the secondary LSP with ‘standby’, we are instructing the Tiberius router to pre-signal this path. Without this keyword, the Tiberius router would only start to signal this second path after the primary path failed.
</p>
<p>
Whenever there is a failure causing a router to take the secondary LSP, the router will try to revert back to the primary. By default, it will do so after the primary LSP has been up (and stable) for 60 seconds. If you don’t care for the router to switch back to a primary path, you can disable this revert-timer by using the following command:
</p>
<pre style="font-size:12px">
set protocols mpls revert-timer 0
</pre>
<p>
To verify that this configuration has taken effect, we can issue the following show command:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show mpls lsp ingress detail</b>
Ingress LSP: 1 sessions

1.1.1.4
  From: 1.1.1.9, State: Up, ActiveRoute: 0, LSPname: L2circuit
  ActivePath: via_Hadrian (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  Revert timer: 0
 <font color='red'>*Primary   via_Hadrian      State: Up</font>
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 40)
 2.0.0.33 S 2.0.0.42 S 2.0.0.66 S 2.0.0.21 S
    Received RRO :
          2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.21
  <font color='red'>Standby   via_Romulus      State: Up</font>
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 40)
 2.0.0.9 S 2.0.0.49 S 2.0.0.57 S 2.0.0.54 S
    Received RRO :
          2.0.0.9 2.0.0.49 2.0.0.57 2.0.0.54
Total 1 displayed, Up 1, Down 0
</pre>
<p>
Alternatively, you can see the same information using rsvp show commands;
</p>

<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show rsvp session ingress name L2circuit detail</b>
Ingress RSVP: 4 sessions

1.1.1.4
  From: 1.1.1.9, LSPstate: Up, ActiveRoute: 0
  <font color='red'>LSPname: L2circuit, LSPpath: Primary</font>
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 300192
  Resv style: 1 FF, Label in: -, Label out: 300192
  Time left:    -, Since: Wed May 13 12:06:36 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 9379 protocol 0
  PATH rcvfrom: localclient
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 2.0.0.33 (xe-0/2/0.9) 1 pkts
  RESV rcvfrom: 2.0.0.33 (xe-0/2/0.9) 1 pkts
  Explct route: 2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.21
  Record route: <self> 2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.21

1.1.1.4
  From: 1.1.1.9, LSPstate: Up, ActiveRoute: 0
  <font color='red'>LSPname: L2circuit, LSPpath: Secondary</font>
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 300432
  Resv style: 1 FF, Label in: -, Label out: 300432
  Time left:    -, Since: Wed May 13 12:07:04 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 2 receiver 9380 protocol 0
  PATH rcvfrom: localclient
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 2.0.0.9 (xe-0/2/0.3) 1 pkts
  RESV rcvfrom: 2.0.0.9 (xe-0/2/0.3) 1 pkts
  Explct route: 2.0.0.9 2.0.0.49 2.0.0.57 2.0.0.54
  Record route: <self> 2.0.0.9 2.0.0.49 2.0.0.57 2.0.0.54
Total 2 displayed, Up 2, Down 0
</pre>


<p>
Next-up will be activate link-protection and node-link-protection.
</p>
