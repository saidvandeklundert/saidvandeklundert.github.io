---
layout: post
title: Q-in-Q on EX, QFX or VCF
image: /img/juniper_logo.jpg
---


<p>
Configuring Q-in-Q, or dot1q tunneling can lead to some confusion. 
I’ve seen confusion due to changes in the new enhanced Layer 2 CLI configuration and because of a mismatch in Ethertype. 
This is a short article on how QinQ can be configured on an EX, QFX or VCF. 
I’ll configure a dot1q tunnel between two EX4300's and between a VCF and an EX4200.
</p>
<p>
First, using the enhanced Layer 2 CLI, the EX4300 configuration:
</p>

![ Q in Q EX4300](/img/q-in-q-4300.png "Q in Q EX4300") 
                
<pre style="font-size:12px">set interfaces ge-0/0/0 description cpe
set interfaces ge-0/0/0 flexible-vlan-tagging
set interfaces ge-0/0/0 encapsulation extended-vlan-bridge
set interfaces ge-0/0/0 unit 3000 vlan-id-list 1-4094
set interfaces ge-0/0/0 unit 3000 input-vlan-map push
set interfaces ge-0/0/0 unit 3000 output-vlan-map pop

set interfaces ge-0/0/47 description ex4300
set interfaces ge-0/0/47 flexible-vlan-tagging
set interfaces ge-0/0/47 encapsulation extended-vlan-bridge
set interfaces ge-0/0/47 unit 3000 vlan-id 3000


set vlans q-in-q interface ge-0/0/0.3000
set vlans q-in-q interface ge-0/0/47.3000</pre>    

<p>
    This setup will switch traffic between the CPEs just fine. 
    All the VLANs specified in the ‘vlan-id-list’  can be used on the CPE devices. 
    The EX4300s will tunnel these frames by placing VLAN tag 3000 in front of any of the tags used by the customer. 
    The above configuration can be used on VCF and QFX as well.
</p>
<p>
Now, we'll look at a scenario where a VCF and an EX4200 are tunneling across a switch that is not configured for QinQ:
</p>


![ Q in Q VCF EX4200](/img/q-in-q-vcf-ex4200.png "Q in Q VCF EX4200") 

<p>
    The configuration I used for the EX4300 was applied to the VCF as well. In addition, the rightmost EX4200 was configured for QinQ like this:
</p>
<pre style="font-size:12px">set interfaces ge-0/0/0 description cpe
set interfaces ge-0/0/0 unit 0 family ethernet-switching port-mode access
set interfaces ge-0/0/0 unit 0 family ethernet-switching vlan members 3000

set interfaces ge-0/0/47 description ex4300
set interfaces ge-0/0/47 unit 0 family ethernet-switching port-mode trunk
set interfaces ge-0/0/47 unit 0 family ethernet-switching vlan members 3000

set vlans q-in-q vlan-id 3000
set vlans q-in-q dot1q-tunneling customer-vlans 1-4094</pre>  
<p>
    There was no connectivity between the CPEs. I realized that the Ethertypes did not match. To fix this, I added the following configuration of the EX4200:
</p>
<pre style="font-size:12px">set ethernet-switching-options dot1q-tunneling ether-type 0x8100</pre>  

<p>
    After issuing this configuration command, both switches that were performing the tunneling started doing so using the same Ethertype.
    Most Juniper switches support TPIDs 0x8100, 0x88a8 and 0x9100. 
    To factor in intermediate switches that do not support or perform QinQ, I figured 0x8100 was the way to go. 
</p>
<p>
    Perhaps a little needless to say, but do not forget to factor in the added VLAN when configuring the MTU.
</p>

<p>
    Ps, to specify the Ethertype using the enhanced Layer 2 CLI:
</p>
<pre style="font-size:12px">set interfaces ge-0/0/47 ether-options ethernet-switch-profile tag-protocol-id 0x8100</pre> 

<br>                

