---
layout: post
title: Policy based LSP mapping with Junos OS
image: /img/juniper_logo.jpg
---

<p>                
LSPs can be configured with a whole variety of characteristics.  You can police traffic that is send onto an LSP, steer the LSP through certain location in the network and much more. When you create several LSPs towards the same destination router, prefixes using that router as a next-hop are randomly divided across those LSPs. What I recently found out is that you can map traffic onto a specific LSP using policies. Exercising control in this way offers some interesting possibilities. Steering or policing entire services by sending them onto a specific LSP is just one of the possibilities.  
</p>   

<p>
Let’s walk through a configuration example where we’ll use the BGP community ‘peer’ for policy based LSP mapping. 
Let’s look at the scenario before any policy is created:
</p>


![ LSP mapping ](/img/juniper-lsp-mapping-scenario.png "LSP mapping") 

<br>  

<p>
The pe1 router has three LSPs towards the pe router. 
All routes that originated from the peer router have a community attached to it. Let’s look at those routes on the pe1 router;
</p>


             
<pre style="font-size:12px">play@pe1> show route community-name peer

inet.0: 426 destinations, 426 routes (426 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

96.172.128.0/19    *[BGP/170] 01:16:47, localpref 150, from 172.16.0.2
                      AS path: 900 12883 12883 12883 49125 I, validation-state: unverified
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path pe1-to-pe-lsp1
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path pe1-to-pe-lsp2
                    > to 10.1.0.1 via xe-0/2/0.1508, label-switched-path <font color='red'>pe1-to-pe-peer</font>
96.172.192.0/19    *[BGP/170] 01:16:47, localpref 150, from 172.16.0.2
                      AS path: 900 28917 6789 48330 I, validation-state: unverified
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path pe1-to-pe-lsp1
                    > to 10.1.0.1 via xe-0/2/0.1508, label-switched-path <font color='red'>pe1-to-pe-lsp2</font>
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path pe1-to-pe-peer
96.172.192.0/24    *[BGP/170] 01:16:47, localpref 150, from 172.16.0.2
                      AS path: 900 28917 6789 48330 I, validation-state: unverified
                    > to 10.1.0.1 via xe-0/2/0.1508, label-switched-path <font color='red'>pe1-to-pe-lsp1</font>
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path pe1-to-pe-lsp2
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path pe1-to-pe-peer
</pre>
                    
<p>
By dividing the prefixes across the available tunnels, the router is displaying the default behavior. To force traffic onto a specific LSP, we can use the action ‘install-nexthop’ in a policy. In our case, we want to match the ‘peer’ community and we want to use the ‘pe1-to-pe-peer’ LSP:
</p>

<pre style="font-size:12px">
play@pe1> show configuration policy-options
policy-statement map-onto-lsp {
	term peer {
		from community peer;
		then {
			install-nexthop lsp pe1-to-pe-peer;
		}
	}
}
</pre>

<p>
We now need to apply this policy to the forwarding table:
</p>

<pre style="font-size:12px">
play@pe1> show configuration routing-options
forwarding-table {
    export map-onto-lsp;
}
</pre>

<p>
After applying this configuration, the pe1 router will start to use only use 1 LSP for prefixes that have the peer community added to it:
</p>
<br>

![ LSP mapping ](/img/juniper-lsp-mapping-scenario-1.png "LSP mapping") 

<p>
To verify our configuration, we can issue the same command as before:
</p>

<pre style="font-size:12px">
play@pe1> show route community-name peer

inet.0: 426 destinations, 426 routes (426 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

96.172.128.0/19    *[BGP/170] 01:15:49, localpref 150, from 172.16.0.2
                      AS path: 900 12883 12883 12883 49125 I, validation-state: unverified
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path <font color='red'>pe1-to-pe-peer</font>
96.172.192.0/19    *[BGP/170] 01:15:49, localpref 150, from 172.16.0.2
                      AS path: 900 28917 6789 48330 I, validation-state: unverified
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path <font color='red'>pe1-to-pe-peer</font>
96.172.192.0/24    *[BGP/170] 01:15:49, localpref 150, from 172.16.0.2
                      AS path: 900 28917 6789 48330 I, validation-state: unverified
                      to 10.1.0.1 via xe-0/2/0.1508, label-switched-path <font color='red'>pe1-to-pe-peer</font>
</pre>

<p>
In this example, I used a policy to match routes based on their community. This is just one of the possibilities. Policy based LSP mapping can also be done by using a policy to match other criteria. Some of these are Route Target Community (L3VPN/VPLS), subnet, CoS values and more. It offers plenty of possibilities to put you in control of traffic and it’s a fun thing to play with.
</p>                
  

