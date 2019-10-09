---
layout: post
title: Juniper Virtual Chassis Fabric
tags: [juniper, qfx, vcf ]
image: /img/juniper_logo.jpg
---

<p>
Having to deal with a network edge that organically grew as time passed, evolving into an ever more complicated constellation of switches, is frustrating.
Looming in the back of my mind were choices made in a past I had no part of. 
Those choices strained growth and frustrated my attempt to keep a clear overview of the network. 
</p>
<p>
In this sense, redoing the network edge with the <b>Virtual Chassis Fabric</b> offered a breath of fresh air in the data center.
I installed and integrated a Virtual Chassis Fabric (VCF) into the network not too long ago. 
The speed and ease with which I was able to deploy the VCF was really impressive. 
It was quite an exciting project, playing with this relatively new architecture. 
After completing the integration, I thought I’d share my reasons for choosing the VCF and show you the basic configuration.
</p>
<br>
<h4>
My reasons for choosing a VCF
</h4>
<br>
<p>
My primary reason for wanting a VCF was definitely the scalability. 
With a VCF, up to 32 member switches can be combined into a single logical switch. This allows a VCF to grow into a logical switch that offers more than 2000 ports. 
</p>
          

![Virtual Chassis Fabric](/img/virtual-chassis-fabric-roles.png "Virtual Chassis Fabric") 

<p>
Switches in a VCF are either leaf or spine. The spine nodes interconnect the leaf nodes. 
The leaf nodes can be regarded as line cards. Whenever you are out of ports, simply add another leaf.
</p>
<p>
I chose to run a mixed mode VCF. 
This allowed me to run both EX4300 and QFX5100 as leaf nodes.
The spine nodes must be QFX5100’s. Running the VCF in mixed mode means that some features are lost.  Since the majority of ports I have in use today are still gigabit, I accepted this. Having access to the more economic EX4300 gigabit ports simply made more sense.
</p>
<p>
Another reason for wanting to run a VCF was the usability of the individual components of the VCF. 
These individual components, especially the QFX5100, allow for a vast amount of architectures: 
</p>


![QFX network architecture options](/img/qfx_options.png "QFX network architecture options") 

<p>
After having invested in a VCF, the individual parts can be used to move into or add some of these other architectures as well.  
This can be done without adding more switches to your inventory.
</p>
<br>
<h4>
The basic VCF configuration.
</h4>
<br>
<p>
Let’s look at a small 4 member mixed-mode VCF example configuration:
</p>
          

![VCF](/img/vcf_example.png "VCF") 

<p>
After placing the hardware and patching the switches, you will need to tell the individual switches on what ports they are connected to the other members.
This can be done by using the following command:
</p>
<pre style="font-size:12px">
request virtual-chassis vc-port set <i>pic-slot pic-slot-number</i> port <i>port-number</i></pre>
<p>
This will turn a regular Ethernet port into a VC-port. I use both 40Gbit and 10Gbit ports as VC-ports. 
</p>
<p>
You should also make sure that the individual switches are running in the proper mode.
Enabling the mixed fabric mode on switches requires the following command:
</p>
                
<pre style="font-size:12px">request virtual-chassis mode fabric mixed local reboot</pre>
<p>
This command will set the mode for the switch on which the command is issued and follows it up with the necessary reboot. 
</p>       
<p>
After this, we move to the <b>[ virtual-chassis ]</b> configuration stanza. 
Issuing the following configuration commands on 1 of the spine nodes is really all there is to it.
</p>
<pre style="font-size:12px">
set virtual-chassis preprovisioned
set virtual-chassis no-split-detection				! recommended for a 2 member configuration 
set virtual-chassis member 0 role routing-engine
set virtual-chassis member 0 serial-number VF3737373737
set virtual-chassis member 1 role routing-engine
set virtual-chassis member 1 serial-number VF3838383838
set virtual-chassis member 2 role line-card
set virtual-chassis member 2 serial-number PE4040404040
set virtual-chassis member 3 role line-card
set virtual-chassis member 3 serial-number PE3939393939</pre>

<p>
Be sure that you do not miss out on the redundancy features that Junos has to offer.
These features will make for a more resilient control plane in case you lose the master routing-engine.
These features will also enable a NonStop Software Upgrade, or NSSU. 
</p>
<p>
The first thing is to activate the redundancy features called GRES, NSR and NSB. These features are configured as follows;
</p>
<pre style="font-size:12px">
set system commit synchronize                                           ! needed for NSR
set chassis redundancy graceful-switchover                              ! GRES
set routing-options nonstop-routing                                     ! NSR
set protocols layer2-control nonstop-bridging                           ! NSB, active only during NSSU
</pre>
<p>
The NSSU upgrade is not something that’s configured. 
It is a feature that relies on the other redundancy features.
The NSSU can be initiated by issuing the following command;
</p>
<pre style="font-size:12px">
request system software nonstop-upgrade set [/var/tmp/install/jinstall-ex-4300-13.2X51-D30.4-domesticsigned.tgz /var/tmp/jinstall-qfx-5-13.2X51-D30.4-domestic-signed.tgz ]</pre>                
<br>
<p>
Covering the reasons for and the configuration of the VCF, I hope this post has given you something to think about. Maybe it gave you some reasons to start thinking about your own VCF. The VCF is still quite new and in constant development. New switches and features will be added in the years to come, so be sure to check out the Juniper website for an up-to-date overview. This way, you can make sure that you won’t miss out on any of the possibilities that the VCF can bring to your network. 
</p>