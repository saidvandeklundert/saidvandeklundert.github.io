---
layout: post
title: Link-protection and node-link-protection on Juniper MX
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---
   
<p>
Protecting LSPs in an MPLS enabled network can save quite some downtime whenever a link or a node in your network fails. 
In this article, we’ll go through the configuration of both <b>link-protection</b> and <b>node-link-protection</b>. We’ll configure it for the following scenario:
</p>

![RSVP node-link protection](/img/node-link-protection-1.png "RSVP node-link protection")                 


<p>
The routers in the network above are running IS-IS and they have MPLS and RSVP enabled for the interfaces that connect them together. 
The configuration is still largely like it was during <a href="2015-05-12-juniper-basic-rsvp-signaled-lsp-mx.md">this</a> post. 
</p>
<h3>
Link-protection:
</h3>
<p>
Before enabling link-protection (or node-link-protection), we have to enable the network to be able to support it. This is done in the <b>[protocols rsvp]</b> stanza under the RSVP-enabled interfaces;
</p>
		

![RSVP node-link protection](/img/node-link-protection-2.png "RSVP node-link protection")      

                            
<p>
Configuring ‘link-protection’ has to be done for all RSVP-enabled interfaces in the network. With this configuration statement, we’re telling the router to signal a bypass LSP as soon as an LSP that wants protection traverses this link.  This protection can be either link-protection or node-link-protection.
</p>
<p>
After this, we can move on to enable link-protection on an LSP. The router already has an LSP configured:
</p>
<pre style="font-size:12px">
Tiberius> <b>show configuration protocols mpls</b>
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
To protect both the primary and the secondary LSP from individual link-failures across the network, we only need to add 1 configuration statement on the ingress LSR:
</p>

![RSVP node-link protection](/img/node-link-protection-3.png "RSVP node-link protection")      

<p>
As soon as this configuration command is added, the Tiberius router will signal to the other routers that the LSP is one that wants link-protection. The other routers, with their RSVP-interfaces enabled for link-protection, will start signaling bypass LSPs where possible.
</p>
<p>
To verify link-protection is enabled, we can check the following on the Tiberius router:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show rsvp session ingress name to_Commodus detail</b>
Ingress RSVP: 4 sessions

1.1.1.4
  From: 1.1.1.9, LSPstate: Up, ActiveRoute: 0
  LSPname: to_Commodus, LSPpath: Primary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 300560
  Resv style: 1 SE, Label in: -, Label out: 300560
  Time left:    -, Since: Fri May 15 08:29:28 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 12472 protocol 0
  <font color='red'>Link protection desired</font>
  Type: Link protected LSP
  PATH rcvfrom: localclient
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 2.0.0.33 (xe-0/2/0.9) 1 pkts
  RESV rcvfrom: 2.0.0.33 (xe-0/2/0.9) 3 pkts
  Explct route: 2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.21
  Record route: &lt;self> 1.1.1.10 (node-id) 2.0.0.33 1.1.1.11 (node-id) 2.0.0.42 1.1.1.12 (node-id) 2.0.0.66 1.1.1.4 (node-id) 2.0.0.21

1.1.1.4
  From: 1.1.1.9, LSPstate: Up, ActiveRoute: 0
  LSPname: to_Commodus, LSPpath: Secondary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 300800
  Resv style: 1 SE, Label in: -, Label out: 300800
  Time left:    -, Since: Fri May 15 08:29:58 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 2 receiver 12473 protocol 0
  <font color='red'>Link protection desired</font>
  Type: Link protected LSP
  PATH rcvfrom: localclient
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 2.0.0.9 (xe-0/2/0.3) 1 pkts
  RESV rcvfrom: 2.0.0.9 (xe-0/2/0.3) 4 pkts
  Explct route: 2.0.0.9 2.0.0.49 2.0.0.57 2.0.0.54
  Record route: &lt;self> 1.1.1.6 (node-id) 2.0.0.9 1.1.1.7 (node-id) 2.0.0.49 1.1.1.8 (node-id) 2.0.0.57 1.1.1.4 (node-id) 2.0.0.54
Total 2 displayed, Up 2, Down 0                    
</pre>
<p>
Or alternatively;
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show mpls lsp name to_Commodus extensive</b>
Ingress LSP: 1 sessions

1.1.1.4
  From: 1.1.1.9, State: Up, ActiveRoute: 0, LSPname: to_Commodus
  ActivePath: via_Hadrian (primary)
  <font color='red'>Link protection desired</font>
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  Revert timer: 0
 *Primary   via_Hadrian      State: Up
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 40)
 2.0.0.33 S 2.0.0.42 S 2.0.0.66 S 2.0.0.21 S
    ...
    &lt;output omitted>
    ...
    <font color='red'>8 May 15 08:30:22.911 Link-protection Up</font>
    ...
    &lt;output omitted>
    ...
    2 May 15 08:29:28.906 Originate Call
    1 May 15 08:29:28.906 CSPF: computation result accepted  2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.21
  Standby   via_Romulus      State: Up
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 40)
 2.0.0.9 S 2.0.0.49 S 2.0.0.57 S 2.0.0.54 S
    ...
    &lt;output omitted>
    ...
    <font color='red'>8 May 15 08:31:42.409 Link-protection Up</font>
    ...
    &lt;output omitted>
    ...
    2 May 15 08:29:58.835 Originate Call
    1 May 15 08:29:58.835 CSPF: computation result accepted  2.0.0.9 2.0.0.49 2.0.0.57 2.0.0.54
                  
</pre>
<p>
We can also move over to the Nero router. Here we can verify that Nero has already pre-signaled a bypass LSP:
</p>

![RSVP node-link protection](/img/node-link-protection-4.png "RSVP node-link protection")      

<p>
Traffic traversing the normal LSP runs through Hadrian, Nero and Septimus before ending up on Commodus. This can be obtained by examining the ‘Record Route’ listed in the output from the ‘show rsvp session ingress name to_Commodus detail’ command. 
</p>
<p>
In that output, we previously saw the following in the Record Route:’ 1.1.1.12 (node-id) 2.0.0.66’ This node-id belongs to Septimus and ‘2.0.0.66’ is configured for one of Septimus’ interfaces.
</p>
<p>
Let’s check Nero for ingress Bypass LSPs:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show rsvp session ingress</b>
Ingress RSVP: 2 sessions
To              From            State   Rt Style Labelin Labelout LSPname
1.1.1.9         1.1.1.11        Up       0  1 SE       -   300768 Bypass->2.0.0.41->2.0.0.34
<font color='red'>1.1.1.12        1.1.1.11        Up       0  1 SE       -   300896 Bypass->2.0.0.66</font>
</pre>
<p>
The appropriately named bypass LSP ‘Bypass->2.0.0.66’  can be used to circumvent the link between Nero and Septimus when it fails. To see more details about this LSP, we can issue to following command:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show rsvp session name Bypass->2.0.0.66 detail</b>
Ingress RSVP: 2 sessions

1.1.1.12
  From: 1.1.1.11, LSPstate: Up, ActiveRoute: 0
  <font color='red'>LSPname: Bypass->2.0.0.66</font>
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 300896
  Resv style: 1 SE, Label in: -, Label out: 300896
  Time left:    -, Since: Fri May 15 08:29:31 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 60214 protocol 0
  Type: Bypass LSP
    Number of data route tunnel through: 1
    Number of RSVP session tunnel through: 0
  PATH rcvfrom: localclient
  Adspec: sent MTU 1500
  Path MTU: received 1500
  PATH sentto: 2.0.0.45 (xe-0/2/0.12) 1 pkts
  RESV rcvfrom: 2.0.0.45 (xe-0/2/0.12) 1 pkts
  Explct route: 2.0.0.45 2.0.0.57 2.0.0.17
  <font color='red'>Record route: &lt;self> 2.0.0.45 2.0.0.57 2.0.0.17</font>
Total 1 displayed, Up 1, Down 0                    
</pre>
<p>
Simply by using the name Bypass->2.0.0.66, you can verify the LSP on the other routers as well. Either through ‘show rsvp session name Bypass->2.0.0.66 detail’ or through ‘show rsvp session name Bypass->2.0.0.66’. 
</p>
<h3>
Node-link-protection:
</h3>
<p>
The last thing to look at is node-link-protection. Commodus has an LSP towards the Tiberius router:
</p>

![RSVP node-link protection](/img/node-link-protection-5.png "RSVP node-link protection")      

<p>
The LSP is configured as follows:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Commodus> <b>show configuration protocols mpls</b>
revert-timer 120;
label-switched-path to_Tiberius {
    to 1.1.1.9;
    ldp-tunneling;
    primary via_Caligula;
    secondary via_Septimus {
        standby;
    }
}
path via_Caligula {
    1.1.1.8 strict;
}
path via_Septimus {
    1.1.1.12 strict;
}
interface xe-0/2/0.14;
interface xe-0/3/0.6;                    
</pre>
<p>
Enabling this LSP for node-link-protection is nothing more than adding the following command:
</p>
<pre style="font-size:12px">
set protocols mpls label-switched-path to_Tiberius node-link-protection                    
</pre>
<p>
Verifying node-link protection hardly differs from verifying link-protection:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Commodus> <b>show rsvp session name to_Tiberius detail</b>
Ingress RSVP: 4 sessions

1.1.1.9
  From: 1.1.1.4, LSPstate: Up, ActiveRoute: 0
  LSPname: to_Tiberius, LSPpath: Primary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 301376
  Resv style: 1 SE, Label in: -, Label out: 301376
  Time left:    -, Since: Wed May 13 21:16:40 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 1 receiver 37581 protocol 0
  <font color='red'><b>Node/Link protection desired</b></font> <font color='0099FF'>&lt;&lt;-instead of the previous '<b>Link protection desired</b>'</font>
  Type: Node/Link protected LSP
    ...
    &lt;output omitted>
    ...


play@MX480-TEST:Commodus> <b>show mpls lsp name to_Tiberius detail</b>
Ingress LSP: 1 sessions

1.1.1.9
  From: 1.1.1.4, State: Up, ActiveRoute: 0, LSPname: to_Tiberius
  ActivePath: via_Caligula (primary)
  <font color='red'>Node/Link protection desired</font>
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
  Revert timer: 120
 *Primary   via_Caligula     State: Up
    ...
    &lt;output omitted>
    ...
</pre>
<p>
Link-node-protection will ask all the transit LSRs to try and establish bypass LSPs around links as well as nodes. A quick look at the Augusts router shows us two bypass LSPs are established:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Augustus> <b>show mpls lsp</b>
Ingress LSP: 0 sessions
Total 0 displayed, Up 0, Down 0

Egress LSP: 2 sessions
To              From            State   Rt Style Labelin Labelout LSPname
1.1.1.7         1.1.1.6         Up       0  1 SE       3        - Bypass->2.0.0.49
1.1.1.7         1.1.1.4         Up       0  1 SE       3        - Bypass->2.0.0.53->2.0.0.58
Total 2 displayed, Up 2, Down 0                    
</pre>
<p>
The first LSP only lists 1 IP address. This is the address of the link that is bypassed. The second LSP lists two IP addresses, indicating multiple links are used for this bypass LSP. This LSP can be used to circumvent an entire node in case a router goes down.
</p>
<p>
One last thing I forgot. Verifying that a link is protected can also be observed through the following command:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show rsvp interface xe-0/3/0.17 extensive</b>
xe-0/3/0.17 Index 361, State Ena/Up
  Authentication, Aggregate, Reliable, <font color='red'>LinkProtection</font>
  HelloInterval 9(second)
  Address 2.0.0.65
  ActiveResv 2, PreemptionCnt 0, Update threshold 10%
  Subscription 100%,
  bc0 = ct0, StaticBW 10Gbps
  ct0: StaticBW 10Gbps, AvailableBW 10Gbps
    MaxAvailableBW 10Gbps = (bc0*subscription)
    ReservedBW [0] 0bps[1] 0bps[2] 0bps[3] 0bps[4] 0bps[5] 0bps[6] 0bps[7] 0bps
  Protection: On, Bypass: 1, LSP: 1, Protected LSP: 1, Unprotected LSP: 0
      3 May 15 08:29:29 New bypass Bypass->2.0.0.66
      2 May 13 12:21:37 Delete bypass Bypass->2.0.0.66->2.0.0.21, inactivity timeout
      1 May 13 09:27:15 New bypass Bypass->2.0.0.66->2.0.0.21
    Bypass: Bypass->2.0.0.66, State: Up, Type: LP, LSP: 1, Backup: 0
      3 May 15 08:29:32 Record Route:  2.0.0.45 2.0.0.57 2.0.0.17
      2 May 15 08:29:32 Up
      1 May 15 08:29:31 CSPF: computation result accepted                    
</pre>
<p>
That’s all there is to it. A little recap picture to sum things up:
</p>

![RSVP node-link protection](/img/node-link-protection-6.png "RSVP node-link protection")      

<p>
Complete configuration is <a href="2015-05-17-link-protection-and-node-link-protection-complete-config.md">here</a>.
</p>