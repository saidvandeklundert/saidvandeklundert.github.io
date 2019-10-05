---
layout: post
title: Basic BGP import filtering example on Junos OS
image: /img/juniper_logo.jpg
---

<p>
	What your BGP peers decide to advertise is out of your control. What you accept is not. 
	This is a short article on basic route-filtering using Junos. The focus here is on a BGP import policy for public peering.
</p>
<p>
	Let’s start of by rejecting all 0.0.0.0/x routes:
</p>

<pre style="font-size:12px">
set policy-options policy-statement reject term default-routes from route-filter 0.0.0.0/0 through 0.0.0.0/32
set policy-options policy-statement reject term default-routes then reject
</pre>
<p>
Here, we match all routes containing exactly '0.0.0.0' with a mask of 0 <b>through</b> 32.
</p>
<p>
 Let’s use <b>prefix-length-range</b> in a second term to filter out all prefixes /25 or longer:
</p>

<pre style="font-size:12px">
set policy-options policy-statement reject term 25-plus from route-filter 0.0.0.0/0 prefix-length-range /25-/32
set policy-options policy-statement reject term 25-plus then reject
</pre>
                
<p>
The 0.0.0.0/0 matches any prefix and by adding the /25-/32 part, we can tell the router to narrow it down to certain prefix-lengths.
</p>
<p>
Another thing we can do without are bogons (routes that have no place in the Internet routing table). 
One way of rejecting those addresses is by using a prefix-list, like this:
</p>
             
<pre style="font-size:12px">
set policy-options policy-statement reject term unwanted from prefix-list-filter bogons orlonger
set policy-options policy-statement reject term unwanted then reject

set policy-options prefix-list bogons 100.64.0.0/10
set policy-options prefix-list bogons 101.10.0.0/19
set policy-options prefix-list bogons 127.0.0.0/8
set policy-options prefix-list bogons 169.254.0.0/16
set policy-options prefix-list bogons 192.0.0.0/24
set policy-options prefix-list bogons 192.0.2.0/24
set policy-options prefix-list bogons 198.18.0.0/15
set policy-options prefix-list bogons 198.51.100.0/24
set policy-options prefix-list bogons 203.0.113.0/24
set policy-options prefix-list bogons 224.0.0.0/4
set policy-options prefix-list bogons 10.0.0.0/8
set policy-options prefix-list bogons 172.16.0.0/12
set policy-options prefix-list bogons 192.168.0.0/16
</pre>
<p>
Using the ‘<b>prefix-list-filter</b>’ statement allows us to use the ‘<b>orlonger</b>’ argument.
The ‘orlonger’ statement will match the prefix referenced, or any prefix that falls within that range.
In this case, the 10.0.0.0/8 entry matches both 10.0.1.0/24 and 10.0.0.1/32.
</p>
<p>
We can also use the previous approach to filter out our own prefixes:
</p>                
<pre style="font-size:12px">
set policy-options policy-statement reject term own-as from prefix-list-filter our-prefixes orlonger
set policy-options policy-statement reject term own-as then reject

set policy-options prefix-list our-prefixes x.x.x.x/x
</pre>

<p>
Let’s have a look at the following scenario before activating our policy:
</p>

![BGP example filter](/img/bgp_example_filter.png "BGP example filter") 

<pre style="font-size:12px">
play@pe> show configuration protocols bgp group peering
import ix;
neighbor 1.1.1.2 {
    peer-as 500;
}
play@pe1> show configuration policy-options policy-statement ix
term accept {
    then accept;
}
</pre>
<p>
The ix policy simply accepts all addresses. Let’s see what the peer is advertising:
</p>                
<pre style="font-size:12px">
play@pe> show route receive-protocol bgp 1.1.1.2

inet.0: 459 destinations, 838 routes (459 active, 0 holddown, 0 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 0.0.0.0/0               1.1.1.2                                 500 I
* 0.0.0.0/2               1.1.1.2                                 500 I
* 0.0.0.0/5               1.1.1.2                                 500 I
* <font color='red'>1.1.0.0/24</font>              1.1.1.2                                 500 I
* 10.0.1.0/24             1.1.1.2                                 500 I
* 10.2.0.0/16             1.1.1.2                                 500 I
* 100.64.1.0/24           1.1.1.2                                 500 I
* 101.10.1.0/24           1.1.1.2                                 500 I
* 169.254.0.0/16          1.1.1.2                                 500 I
* 169.254.0.0/18          1.1.1.2                                 500 I
* 192.168.1.0/24          1.1.1.2                                 500 I
* 224.0.0.0/4             1.1.1.2                                 500 I
</pre>
<p>
The only valid prefix is the <b>1.1.0.0/24</b> prefix. Let’s insert the <b>reject</b> policy prior to the policy already active using the following configuration command:
</p>                
<pre style="font-size:12px">
insert protocols bgp group peering import reject before ix
</pre>                            
<p>
The configuration now looks like this:
</p>
<pre style="font-size:12px">
play@pe> show configuration protocols bgp group peering
import [ reject ix ];
neighbor 1.1.1.2 {
    peer-as 500;
}
</pre>
                
<p>
Because the reject policy appears first in the configuration, it’s evaluated before the ix policy. 
And since the reject policy only ends with terminating action for unwanted addresses, other prefixes will still be evaluated by the ix policy. 
To verify the result of our configuration:
</p>
<pre style="font-size:12px">
play@pe> show route receive-protocol bgp 1.1.1.2

inet.0: 459 destinations, 856 routes (459 active, 0 holddown, 11 hidden)
  Prefix                  Nexthop              MED     Lclpref    AS path
* 1.1.0.0/24              1.1.1.2                                 500 I
</pre>                
                
<p>
The 11 routes we didn’t want are now hidden (not used because of a routing policy).
</p>
                
<p>
The nice thing is that this basic import filter rejects routes we wouldn’t want to receive anywhere so we can use it everywhere.
You’ll only have one filter to keep updated across all places where you peer.
The bogon list for IPv6 is a bit lengthier, but a similar approach could be used to handle IPv6 peers.                       
And just so you know, other Junos matching conditions are described <a href="http://www.juniper.net/techpubs/en_US/junos12.3/topics/usage-guidelines/policy-configuring-route-lists-for-use-in-routing-policy-match-conditions.html" target="_blank">here</a>.                    
</p>                

<p>
PS.
the bogon list is not a static list. 
Find a reliable source that can provide you with an updated list of bogons, ‘<a href="http://www.team-cymru.org/bogon-reference.html" target="_blank">Team CYMRU</a>’ for example. 
</p>                

     


