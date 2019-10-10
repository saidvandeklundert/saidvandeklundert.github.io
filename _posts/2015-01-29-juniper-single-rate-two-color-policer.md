---
layout: post
title: Single-rate two-color policer on an EX.
tags: [juniper, cos]
image: /img/juniper_logo.jpg
---


<p>
Policing, also known as rate-limiting, can be used as an instrument to control how much traffic is allowed to flow in a certain direction. 
In Juniper, you can do this by using a policer as an action in a firewall filter. 
This article is about the configuration of two simple and straightforward examples involving a policer on a Juniper device that is referenced in a firewall filter. 
Both examples are performed on an EX4200 and both use the most common or standard policer; the single-rate two-color policer. 
This policer allows for both hard and soft policing meaning that traffic exceeding the policer can be dropped or remarked. 
</p>
<p>
The policer is made up out of two rates. 
These are the CIR (Committed Information Rate) and the CBS (Committed Burst Size). 
The CIR, which is expressed in bits per second, is the guaranteed rate. 
The CBS, expressed in bytes, is amount of burst traffic that is allowed to exceed the guaranteed rate. 
</p>
<p>
As far as the burst size is considered, Juniper offers the following burst size recommendation:<br>
interface bandwidth * allowable time for burst traffic / 8<br>
<br>
Juniper suggests to take a minimum of 5ms as a starting point. 
</p>
<p>
As an alternative to this method, Juniper suggests that you could also opt to multiply the MTU by at least 10.
</p>
<p>
Setting the correct burst size is important. 
This is because a burst size that is too big will not police anything. 
Alternatively, a burst size set to small might be too aggressive and police the traffic rate down too much.
Let’s take a look at a simple example. Suppose that you want to police a certain vlan on port ge-0/0/0 to 50Mbit, like this:
</p>
  

![Juniper policer](/img/juniper_policer_example_1.png "Juniper policer") 

<p>
    First, we would need to obtain the burst size. 
    Since the interface bandwidth is 1 gigabit, the recommended burst size would be ‘1000000000 * 0.005 / 8 = 625.000’. 
    We can then start by configuring our policer, like this:
</p>
<pre style="font-size:12px">  
set firewall policer 50m if-exceeding bandwidth-limit 50m
set firewall policer 50m if-exceeding burst-size-limit 625k
set firewall policer 50m then discard
</pre>    
<p>
    As an alternative, and depending on the platform you are using, you could choose to use a percentage instead of the 50m by using ‘if-exceeding bandwidth-percent’. 
    This is not supported on the EX4200 (on MX or QFX it is).
</p>
<p>
    In this example, we only want to police traffic coming from vlan 1012. 
    In order to apply the policer to inbound traffic in that vlan only, we need to reference the policer in a firewall filter. 
    On an EX switch, we could use something like this:
</p>
<pre style="font-size:12px">  
set firewall family ethernet-switching filter EXAMPLE term POLICE-VLAN1012 from dot1q-tag 1012
set firewall family ethernet-switching filter EXAMPLE term POLICE-VLAN1012 then accept
set firewall family ethernet-switching filter EXAMPLE term POLICE-VLAN1012 then policer 50m
set firewall family ethernet-switching filter EXAMPLE term ALLOW-ALL-ELSE then accept
</pre>  
<p>
    The ‘ALLOW-ALL-ELSE’ term is configured to let all the other traffic pass. 
    Without the term, only traffic in vlan 1012 would be allowed. 
    After this, only one thing remains, which is applying the firewall filter in the inbound direction:
</p>
<pre style="font-size:12px">                 
set interfaces ge-0/0/1 unit 0 family ethernet-switching filter input EXAMPLE
</pre>  
<p>
    On the EX switch (or at least the low end EX-switches) the inbound direction is the only direction in which you can apply a policer.
</p>
<p>
    Let’s look at another EX example. 
    Contrary to MX-series routers, the EX switches do not have a native ARP-rate limiter. 
    You might want to consider limiting ARP traffic on your access-switches to prevent them from being overrun in case something goes wrong in a customer’s network or in a part of your own network. 
    If and when there is a loop somewhere in a network, you can end up having your switch exposed to 200.000+ arp-requests per second. 
    Since ARP requests are broadcasted to initiate communication by hosts, an ARP storm is not that uncommon when there is a loop somewhere.
</p>
<p>
    After you have this filter for ARP in place, you can easily extend it with additional terms. 
    This can be done in case you want to impose further traffic restrictions. 
    You can add terms to restrict traffic towards prefixes, destination/sources ports, etc.
</p>
<p>
    Let’s look at another example on an EX4200. 
    On the following access-port, I will configure a firewall filter that will police ARP requests as well rate-limit traffic towards a certain destination IP:
</p>
    

![Juniper policer](/img/juniper_policer_example_2.png "Juniper policer") 

<pre style="font-size:12px">                 
set firewall policer RATE-LIMIT-ARP if-exceeding bandwidth-limit 32k
set firewall policer RATE-LIMIT-ARP if-exceeding burst-size-limit 1250
set firewall policer RATE-LIMIT-ARP then discard
set firewall policer tcp-host if-exceeding bandwidth-limit 2m
set firewall policer tcp-host if-exceeding burst-size-limit 15k
set firewall policer tcp-host then discard

set firewall family ethernet-switching filter ACCESS-PORT-FILTER-GE-0/0/1 term ARP from ether-type arp
set firewall family ethernet-switching filter ACCESS-PORT-FILTER-GE-0/0/1 term ARP then policer RATE-LIMIT-ARP
set firewall family ethernet-switching filter ACCESS-PORT-FILTER-GE-0/0/1 term TCP-HOST from destination-address 192.168.51.76/32
set firewall family ethernet-switching filter ACCESS-PORT-FILTER-GE-0/0/1 term TCP-HOST from protocol tcp
set firewall family ethernet-switching filter ACCESS-PORT-FILTER-GE-0/0/1 term TCP-HOST  then policer tcp-host
set firewall family ethernet-switching filter ACCESS-PORT-FILTER-GE-0/0/1 term END-ACCEPT-ALL then accept

set interfaces ge-0/0/1 description some-access-port
set interfaces ge-0/0/1 ether-options no-auto-negotiation
set interfaces ge-0/0/1 ether-options link-mode full-duplex
set interfaces ge-0/0/1 ether-options speed 1g
set interfaces ge-0/0/1 unit 0 family ethernet-switching port-mode access
set interfaces ge-0/0/1 unit 0 family ethernet-switching vlan members 1267
set interfaces ge-0/0/1 unit 0 family ethernet-switching filter input ACCESS-PORT-FILTER-GE-0/0/1

set interfaces vlan unit 1267 description some-vlan-interface
set interfaces vlan unit 1267 family inet address 192.168.1.1/24

set vlans some-vlan vlan-id 1267
set vlans some-vlan l3-interface vlan.1267
</pre>  
<p>
    By configuring individual firewall filters for ports, the printout of the show firewall command becomes very clear as well;
</p>                
<pre style="font-size:12px">   
EX4200-TEST> show firewall

Filter: ACCESS-PORT-FILTER-GE-0/0/42
Policers:
Name                                                Bytes              Packets
RATE-LIMIT-ARP-ARP                                                           0
tcp-host                                                                   3419
</pre>  
<p>
    In these example, I applied two firewall filters to layer 2 interfaces with the ether-switching family configured. 
    As an alternative to this, you can also configure firewall filters referencing a policer on a layer 3 interface (on both a routed-port and a vlan-interface) and you can configure a firewall filter with a policer for a vlan.
</p>
<p>
    One last note.
    A policer in Junos will operate in term-specific mode by default. 
    This means that Junos OS will create an instance of a referenced policer for every individual term. 
    Under the [firewall policer] configuration stanza on an EX switch, you can find the ‘filter-specific’ knob. 
    Activating this, will make the EX switch police multiple terms in the same firewall filter using the same policer.
</p>




