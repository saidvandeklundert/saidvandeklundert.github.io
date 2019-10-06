---
layout: post
title: Using apply-path in a prefix-list on Juniper.
tags: [juniper]
image: /img/juniper_logo.jpg
---
                   
<p>
    Juniper's Junos offers a lot of flexibility as well as nifty little tricks. 
    I recently ran into the situation in which 'apply-path' really came in handy. 
    For a particular service, a different subnet was provisioned under the same interface over and over again. 
    I wanted to advertise all of the prefixes configured on subinterfaces under this interface through BGP. 
    At the same time, however, I did not want to advertise all the subnets that were configured on the router. 
    Another thing I wanted to avoid was having to alter the prefix-list for every future subnet configured for that particular service. 
    To this end, I configured the following prefix list:
</p>
<pre style="font-size:12px">
set policy-options prefix-list direct-xe-2/0/2 apply-path "interfaces xe-2/0/2 unit <*> family inet address <*>"
</pre>     
<p>
    The previous prefix list is basically a dynamically generated prefix list. 
    Junos builds it for you based on what is configured under the interface. 
    There will be a match on all the IP addresses configured only under the referenced interface. 
    With this prefix-list, all future additions will also be automatically added by Junos. To see what the prefix-list is referencing, you can issue the following command:
</p>
<pre style="font-size:12px">
play@MX104> show configuration policy-options prefix-list direct-xe-2/0/2 |display inheritance
##
## apply-path was expanded to:
##     1.1.1.1/29;
##
apply-path "interfaces xe-2/0/2 unit <*> family inet address <*>";
</pre>   
<p>
    The only thing left after this was to reference the prefix-list in a policy term:
</p>
<pre style="font-size:12px">
set policy-options policy-statement BGP-EXPORT term DIRECT from family inet
set policy-options policy-statement BGP-EXPORT term DIRECT from prefix-list direct-xe-2/0/2
set policy-options policy-statement BGP-EXPORT term DIRECT then next-hop self
set policy-options policy-statement BGP-EXPORT term DIRECT then accept
</pre>      
<p>
    This is only one of the many situations in which this feature can come in handy. 
    For instance, the same trick can be used to allow incoming connections to the router itself.
    What if you wanted to allow incoming BGP requests only for the IP addresses that are configured to actually be a BGP neighbor?
</p>
<pre style="font-size:12px">
set policy-options prefix-list bgp-re-allow apply-path "protocols bgp group <*> neighbor <*>"
</pre>     
<p>
    You can then proceed to reference this prefix list in a firewall term under the filter applied to lo0.
    Anyway, these are just two simple examples.  Many more is possible in Junos.
</p>
   