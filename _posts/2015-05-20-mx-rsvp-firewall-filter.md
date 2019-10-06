---
layout: post
title: MX RSVP firewall filter
tags: [juniper]
image: /img/juniper_logo.jpg
---

<p>
Of course, you need to allow RSVP in the firewall filter you are using to protect the routing-engine. 
The book 'Juniper MX series' covers this very in-depth in chapter 4. 
It offers a very extensive guide or example on how you could go about building a proper firewall filter to protect the RE.
</p>
<p>
    Basically, what the book suggests is to use an input-list of several filters that is applied to the lo0 interface:
</p>
<br>                

![juniper firewall filter](/img/firewall-filter.png "Juniper firewall filter") 

<p>
    These individual filters are filters that consist of terms allowing individual protocols, like this:
</p>
<br>                

![juniper firewall filter](/img/firewall-filter-2.png "Juniper firewall filter")   

<br>   
<p>
	In my opinion, configuring RE-protection in this way is very nice and clean.
</p>
<p>
	Anyway, I used this as a starting point when I needed to define filters for my network. 
	However, when I applied the firewall filter, I ran into a problem. I checked and verified that the configured LSPs were up. 
	Upon doing so, I also happened to notice that there were no transit LSPs active on the MX. 
	This was very strange; I would expect to see a few detours at least. 
</p>
<p>
    In the book, the example filter goes like this:
</p>
<pre style="font-size:12px">
set firewall filter accept-rsvp term accept-rsvp from destination-prefix-list router-ipv4
set firewall filter accept-rsvp term accept-rsvp from protocol rsvp
set firewall filter accept-rsvp term accept-rsvp then accept                    
</pre>
<p>
    The ‘router-ipv4’ prefix-list is configured like this:
</p>
<pre style="font-size:12px">
set policy-options prefix-list router-ipv4 apply-path "interfaces <*> unit <*> family inet address <*>"
</pre>
<p>
    This prefix-list referenced in ‘accept-rsvp’ will inherit all the subnets that are configured on the interfaces of the router. 
    This will cause a problem for other routers when they want an LSP to transit the router on which the filter is configured.                     
</p>
<p>
	RSVP signaled LSPs are targeted to the LSP end-point and use the destination IP address of the egress router of the LSP.  
	The ‘router-alert’ option in IP makes the routers along the path inspect the packet.
	The transit routers that the RSVP messages pass will check that destination IP address with its configured firewall filters.
	Initially, the configured firewall filter only accepted IP addresses configured on the router itself. Therefore, the MX dropped the message.
	For this reason, I did not see any transit LSPs traversing the MX.
</p>
<p>
    A small change to the firewall filter fixed the issue. 
	First, I configured a prefix list that contained all the loopback IP addresses of the routers running RSVP in the network. For example;
</p>
<pre style="font-size:12px">
set policy-options prefix-list core-loopback 10.0.0.0/24
set policy-options prefix-list core-loopback 10.0.1.0/24
set policy-options prefix-list core-loopback 10.0.2.0/24                    
</pre>
<p>
    Then, I used that prefix list as a destination prefix list in the 'accept-rsvp' filter:
</p>
<pre style="font-size:12px">
set firewall filter accept-rsvp term accept-rsvp from destination-prefix-list core-loopback
set firewall filter accept-rsvp term accept-rsvp from protocol rsvp
set firewall filter accept-rsvp term accept-rsvp then accept                    
</pre>
<p>
    The RSVP filter was added to the routing-protocols I wanted to permit:
</p>
<pre style="font-size:12px">
set firewall filter accept-routing term accept-rsvp filter accept-rsvp
</pre>
<p>
    And the filters were applied to the lo0 interface:
</p>
<pre style="font-size:12px">
set interfaces lo0 unit 0 family inet filter input-list accept-services
set interfaces lo0 unit 0 family inet filter input-list accept-routing
set interfaces lo0 unit 0 family inet filter input-list discard-all                    
</pre>
<p>
    Anyway, if you happen to be working with MX routers, you owe it to yourself to read this book. 
	It is the best book on Juniper equipment I have ever read.
	When I started working for a company that ran a Juniper MX core, I read it and found it to be invaluable. It’s simply superb.
</p>

![juniper MX Series](/img/Juniper-mx-series.gif "Juniper MX Series")  

<p>
                    
<b>Juniper MX Series</b><br>
<br>                    
By: Douglas Richard Hanks Jr., Harry Reynolds<br>
</p>

<p>
Note,<br>
if you’re thinking about using the filters from the book, I remember 1 other thing I ran into as well. 
This was with the prefix-lists used for BGP. I ended up using the following prefix-lists for BGP:
</p>
<pre style="font-size:12px">
set policy-options prefix-list bgp-neighbor-v4 apply-path "protocols bgp group <*> neighbor <*.*>"
set policy-options prefix-list bgp-neighbor-v6 apply-path "protocols bgp group <*> neighbor <*:*>"                
</pre>
<p>
    This will neatly separate you’re IPv4 and IPv6 neighbors into two separate prefix-lists.
</p>
