---
layout: post
title: Fast reroute
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---

<p>
Traffic sent across RSVP-signaled LSPs without any additional configuration is susceptible to quite some down-time when a node or a link in the network fails.
In a previous article <a href="https://saidvandeklundert.net/2015-05-13-juniper-primary-and-secondary-lsp/">here</a>, I made an LSP more robust by configuring  a primary and a secondary LSP.
Let’s further enhance the LSP by configuring and verifying fast-reroute (FRR).
</p>      
<p>
Our starting position is an LSP from Tiberius to Commodus:
</p>

![FRR](/img/fast-reroute-rsvp-frr-1.png "FRR") 

<p>
The LSP already has a primary and a secondary path. Let’s look at the current MPLS configuration:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show configuration protocols mpls</b>
revert-timer 0;
label-switched-path to_Commodus {
    to 1.1.1.4;
    ldp-tunneling;
    primary via_Hadrian;
    secondary via_Romulus {
        standby;
    }
}
path via_Romulus {
    1.1.1.6 strict;
}
path via_Hadrian {
    1.1.1.10 strict;
}
interface xe-0/2/0.9;
interface xe-0/2/0.3;                    
</pre>
<p>
Configuring FRR is nothing more than adding the following statement to the label-switched-path on the ingress router (Tiberius):
</p>
<pre style="font-size:12px">
set protocols mpls label-switched-path to_Commodus fast-reroute                    
</pre>
<p>
This will make the ingress LSR add an object to the RSVP path messages that it sends out. 
Through this object, it is basically requesting all transit LSRs encountered along the LSP to try and signal for detours. Imagine that the link between Nero and Septimus breaks:
</p>

![FRR](/img/fast-reroute-rsvp-frr-2.png "FRR") 


<p>
As soon as Nero sees that the link is broken, it will do two things. 
Nero will send a PathErr message to the ingress LSR (Tiberius).
Additionally, since the LSP is now broken, the Nero router will immediately start using the detour (red line). This way, it can still deliver traffic that is currently being received across the LSP. This as opposed to black holing it.
</p>
<p>
The nice thing is that these detours are available immediately. The bad thing is that they can serve only 1 LSP. Every LSP configured for FRR will have transit routers signal their own and individual detour LSPs. 
</p>
<p>
Back to the example, let’s verify that the LSP is now configured for FRR:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show mpls lsp name to_Commodus detail</b>
Ingress LSP: 1 sessions

1.1.1.4
  From: 1.1.1.9, State: Up, ActiveRoute: 0, LSPname: to_Commodus
  ActivePath: via_Hadrian (primary)
  <font color='red'>FastReroute desired</font>
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  Revert timer: 0
 *Primary   via_Hadrian      State: Up
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 40)
 2.0.0.33 S 2.0.0.42 S 2.0.0.66 S 2.0.0.21 S
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          2.0.0.33(flag=9) 2.0.0.42(flag=9) 2.0.0.66(flag=1) 2.0.0.21
  Standby   via_Romulus      State: Up
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 40)
 2.0.0.9 S 2.0.0.49 S 2.0.0.57 S 2.0.0.54 S
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          2.0.0.9(flag=9) 2.0.0.49(flag=9) 2.0.0.57(flag=1) 2.0.0.54
Total 1 displayed, Up 1, Down 0                    
</pre>
<p>
The previous command is an indication that the ingress LSR, Tiberius, is indicating to other routers that this path should be protected by FRR. The other routers, as long as they understand the RSVP object for FRR and as long as the topology permits them, will now seek to establish detours.
</p>
<p>
The fact that FRR was requested is also visible on the transit LSRs. Let’s look at the LSPs primary path on the Nero router:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show rsvp session transit name to_Commodus detail</b>
Transit RSVP: 6 sessions, 1 detours

1.1.1.4
  From: 1.1.1.9, LSPstate: Up, ActiveRoute: 0
  LSPname: to_Commodus, LSPpath: Primary
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 300352
  Resv style: 1 FF, Label in: 302112, Label out: 300352
  Time left:  122, Since: Wed May 13 23:29:49 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 5 receiver 12470 protocol 0
  <font color='red'>FastReroute desired</font>
  PATH rcvfrom: 2.0.0.41 (xe-0/2/0.11) 12 pkts
  Adspec: received MTU 1500 sent MTU 1500
  PATH sentto: 2.0.0.66 (xe-0/3/0.17) 6 pkts
  RESV rcvfrom: 2.0.0.66 (xe-0/3/0.17) 3 pkts
  Explct route: 2.0.0.66 2.0.0.21
  Record route: 2.0.0.34 2.0.0.41 &lt;self> 2.0.0.66 2.0.0.21
    Detour is Up
    Detour Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
    Detour adspec: received MTU 1500 sent MTU 1500
    Path MTU: received 1500
    Detour PATH sentto: 2.0.0.45 (xe-0/2/0.12) 5 pkts
    Detour RESV rcvfrom: 2.0.0.45 (xe-0/2/0.12) 2 pkts
    <font color='red'>Detour Explct route: 2.0.0.45 2.0.0.57 2.0.0.54</font>
    Detour Record route: 2.0.0.34 2.0.0.41 &lt;self> 2.0.0.45 2.0.0.57 2.0.0.54
    Detour Label out: <font color='red'>300848</font>                    
</pre>
<p>
FRR was requested and a detour was granted. The ‘Detour Explct route’ part is the physical path that the detour takes. 
These IP addresses belong to Augustus, Caligula and Commodus. So in this case, the detour will skip the Septimus node completely. 
</p>
<p>
The detour LSP is pre-signaled, so in this case, we will be able to see the detour LSP on the Augustus router as well:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Augustus> <b>show rsvp session transit name to_Commodus detail</b>
Transit RSVP: 6 sessions, 1 detours

1.1.1.4
  From: 1.1.1.9, LSPstate: Up, ActiveRoute: 0
  LSPname: to_Commodus, LSPpath: Primary
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 301472
  Resv style: 1 FF, Label in: 300864, Label out: 301472
  Time left:  123, Since: Wed May 13 23:29:52 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 5 receiver 12470 protocol 0
  Detour branch from 2.0.0.37, to skip 1.1.1.11, Up
    Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
    Adspec: received MTU 1500
    Path MTU: received 0
    PATH rcvfrom: 2.0.0.50 (xe-0/3/0.13) 9 pkts
    Adspec: received MTU 1500 sent MTU 1500
    PATH sentto: 2.0.0.57 (xe-0/2/0.15) 1 pkts
    RESV rcvfrom: 2.0.0.57 (xe-0/2/0.15) 3 pkts
    Explct route: 2.0.0.57 2.0.0.54
    Record route: 2.0.0.34 2.0.0.37 2.0.0.50 &lt;self> 2.0.0.57 2.0.0.54
    Label in: 300864, Label out: 301472
  <font color='red'>Detour branch from 2.0.0.46, to skip 1.1.1.12, Up</font>
    Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
    Adspec: received MTU 1500
    Path MTU: received 0
    PATH rcvfrom: 2.0.0.46 (xe-0/3/0.12) 12 pkts
    Adspec: received MTU 1500
    PATH sentto: 2.0.0.57 (xe-0/2/0.15) 0 pkts
    RESV rcvfrom: 2.0.0.57 (xe-0/2/0.15) 0 pkts
    Explct route: 2.0.0.57 2.0.0.54
    Record route: 2.0.0.34 2.0.0.41 2.0.0.46 &lt;self> 2.0.0.57 2.0.0.54
    Label in: 300848, Label out: 301472                    
</pre>
<p>
As soon as Nero detect a problem, it can immediately start using the detour label and the Augustus node will know what to do with it.
</p>
<p>
FRR is extremely easy to configure, but the main caveat is the lack of scalability. Every LSR will need signal a detour for all the LSPs that transit the router. In this scenario, FRR is requested for both the primary and the secondary route. Since all but the egress LSR can generate a detour, this single LSP will lead to about 6 or so detours. Imagine activating FRR for 500 LSPs in a larger network with far more transit LSRs per LSP. Better get into link-protection and node-link-protection fast if you have a big network.
</p>



























