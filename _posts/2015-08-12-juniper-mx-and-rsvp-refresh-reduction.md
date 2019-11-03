---
layout: post
title: Juniper MX and RSVP refresh reduction
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---


<p>
The past few weeks I have been working on the replacement of several core nodes. 
After finally installing the last MX, I wanted revise several configurations that were applied.
One of the configurations that I revised was the configuration used in the RSVP stanza. I ‘optimized’ it by implementing <b>RSVP refresh reduction</b>.
</p>
<p>
RSVP refresh reduction is all about making RSVP less chatty, more scalable and more reliable. 
A detailed explained can be found in RFC 2961 and a summary is given in the ‘<b>Junos OS RSVP Configuration Guide</b>’. 
Here it states that enabling RSVP refresh reduction includes the following features:
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; RSVP message bundling using the bundle message
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; RSVP Message ID to reduce message processing overhead                    
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; Reliable delivery of RSVP messages using Message ID, Message Ack, and Message Nack
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; Summary refresh to reduce the amount of information transmitted every refresh interval
</p>
<p>
The ‘Day One: Deploying MPLS’ also covers RSVP refresh reduction. 
What they describe is that without the refresh reduction, sustaining 7500 LSPs in their network required on average <u>235</u> RSVP messages per second. 
With RSVP refresh reduction enabled, 235 was reduced to <u>12</u> RSVP messages per second!
</p>
<p>
Well now, why wouldn’t I want that? Because it is a hassle to configure and a reboot is required? No way, RSVP refresh reduction requires only two configuration statements and it can be enabled without any impact whatsoever.
</p>
<p>
To configure RSVP refresh reduction, you must add the aggregate and reliable statement to all the interfaces configured under the RSVP stanza, that’s all. Let’s have a look at an example;
</p>

![RSVP refresh reduction](/img/rsvp-refresh-reduction.png "RSVP refresh reduction") 

<p>
Two routers are running ISIS and have several RSVP-signaled LSPs enabled. Enabling RSVP refresh reduction on the routers takes nothing more than the following;
</p>
                
<b>Tiberius:</b>
<pre style="font-size:12px">set protocols rsvp interface xe-0/2/0.9 <font color='red'>aggregate</font>
set protocols rsvp interface xe-0/2/0.9 <font color='red'>reliable</font></pre> 

                
<b>Hadrian:</b>
<pre style="font-size:12px">set protocols rsvp interface xe-0/3/0.9 <font color='red'>aggregate</font>
set protocols rsvp interface xe-0/3/0.9 <font color='red'>reliable</font>
</pre> 
<p>
As soon as these commands are added, the routers will start to signal their RSVP refresh reduction ability by enabling the RR capable bit in the RSVP common header. 
This bit is significant between the two neighbors only so you'll have to configure the entire network link-by-link in this way.
</p>
<p>
Before the activation of RSVP RR:
</p>
                
<pre style="font-size:12px">play@MX480-TEST-RE0:Tiberius> show rsvp neighbor detail
RSVP neighbor: 2 learned
Address: 2.0.0.33 via: xe-0/2/0.9 status: Up
  Last changed time: 6d 18:13:12, Idle: 0 sec, Up cnt: 3, Down cnt: 2
  Message received: 386732
  Hello: sent 178198, received: 178198, interval: 9 sec
  Remote instance: 0x9053dc12, Local instance: 0x9ab7bb10
  Refresh reduction:  <font color='red'>incomplete</font>
    Remote end: disabled, Ack-extension: disabled</pre>                 
<p>
After the activation of RSVP RR:
</p>

                
<pre style="font-size:12px">play@MX480-TEST-RE0:Tiberius> show rsvp neighbor detail
RSVP neighbor: 2 learned

Address: 2.0.0.33 via: xe-0/2/0.9 status: Up
  Last changed time: 6d 18:15:12, Idle: 0 sec, Up cnt: 3, Down cnt: 2
  Message received: 386763
  Hello: sent 178211, received: 178211, interval: 9 sec
  Remote instance: 0x9053dc12, Local instance: 0x9ab7bb10
  Refresh reduction:  <font color='red'>operational</font>
    Remote end: <font color='red'>enabled</font>, Ack-extension: <font color='red'>enabled</font></pre>                 

<p>
Important note; during the activation of RSVP RR, no LSPs were hurt. 
</p>
<p>
Another (possibly better) way to configure the refresh reduction is to configure it using apply-groups.  
RSVP interfaces will most likely be enabled for several options and all of the interfaces will probably be configured in the exact same way. For instance, the configuration for a group that applies node-link protection, authentication and refresh reduction would look like this:
</p>
                
<pre style="font-size:12px">set groups rsvp-interfaces protocols rsvp interface <*> authentication-key "$9$9EGhjHqT3REyvMX/CK8LxsY"
set groups rsvp-interfaces protocols rsvp interface <*> aggregate
set groups rsvp-interfaces protocols rsvp interface <*> reliable
set groups rsvp-interfaces protocols rsvp interface <*> link-protection</pre>       

<p>
After configuring it, you can apply it under the rsvp stanza like this:
</p>
<pre style="font-size:12px">set protocols rsvp <font color='red'>apply-groups rsvp-interfaces</font>
set protocols rsvp interface ge-0/1/0.3001
set protocols rsvp interface ge-0/0/0.3000</pre>    
<p>
To see what the effect is, you can go like this:
</p>

<pre style="font-size:12px">play@MX480-TEST-RE0> show configuration protocols rsvp | display inheritance
interface ge-0/1/0.3001 {
    ##
    ## '$9$9EGhjHqT3REyvMX/CK8LxsY' was inherited from group 'rsvp-interfaces'
    ##
    authentication-key "$9$9EGhjHqT3REyvMX/CK8LxsY"; ## SECRET-DATA
    ##
    ## 'aggregate' was inherited from group 'rsvp-interfaces'
    ##
    aggregate;
    ##
    ## 'reliable' was inherited from group 'rsvp-interfaces'
    ##
    reliable;
    ##
    ## 'link-protection' was inherited from group 'rsvp-interfaces'
    ##
    link-protection;
}
interface ge-0/0/0.3000 {
    ##
    ## '$9$9EGhjHqT3REyvMX/CK8LxsY' was inherited from group 'rsvp-interfaces'
    ##
    authentication-key "$9$9EGhjHqT3REyvMX/CK8LxsY"; ## SECRET-DATA
    ##
    ## 'aggregate' was inherited from group 'rsvp-interfaces'
    ##
    aggregate;
    ##
    ## 'reliable' was inherited from group 'rsvp-interfaces'
    ##
    reliable;
    ##
    ## 'link-protection' was inherited from group 'rsvp-interfaces'
    ##
    link-protection;
}</pre>   
<p>
The main thing isn’t probably even the fact that it saves configuration overhead. By using apply-groups, you are assured that no link will miss out on any of the RSVP configuration statements you intended.
</p>