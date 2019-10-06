---
layout: post
title: QFX5100 802.1Q Tunneling (Q-in-Q)
tags: [juniper]
image: /img/juniper_logo.jpg
---
<p>
    A QFX5100 allows for dot1q-tunneling, or Q-in-Q.
    If you ever configured dot1q-tunneling on an EX-switch, this configuration differs a lot from what you may be used to. 
    This article offers an attempt to clarify and explain the configuration of a dot1q-tunnel on a standalone QFX5100 without an enhanced feature license.
</p>
<p>
    I will use the following setup in the next examples:
</p>

![QFX Q in Q](/img/qfx_qinq_1.png "QFX Q in Q")    

<p>
    What I will do is configure several vlan interfaces on the MX104 routers (Mars and Jupiter) and establish IP connectivity between those interfaces.
</p>
<p>
    Let’s start with the configuration of the QFX5100.  
    To create a dot1q-tunnel between the two ports in the example, I have started with the encapsulation of the interfaces.
</p>
<pre style="font-size:12px">
set interfaces xe-0/0/46 vlan-tagging
set interfaces xe-0/0/46 encapsulation extended-vlan-bridge
set interfaces xe-0/0/47 vlan-tagging
set interfaces xe-0/0/47 encapsulation extended-vlan-bridge
</pre>  
<p>
    After, I applied the interface unit configuration. 
    This configuration involves a vlan input and output map. 
    The input map will push a tag onto the frames send to the QFX, and the output map will simply pop the outer tag. 
</p>
<pre style="font-size:12px">
set interfaces xe-0/0/46 unit 10 input-vlan-map push
set interfaces xe-0/0/46 unit 10 input-vlan-map vlan-id 10
set interfaces xe-0/0/46 unit 10 output-vlan-map pop

set interfaces xe-0/0/47 unit 10 input-vlan-map push
set interfaces xe-0/0/47 unit 10 input-vlan-map vlan-id 10
set interfaces xe-0/0/47 unit 10 output-vlan-map pop
</pre>                    
<p>
    After this, we need to determine what vlans are allowed into the ‘tunnel’. We will start with vlan 1500:
</p>
<pre style="font-size:12px">
set interfaces xe-0/0/46 unit 10 vlan-id-list 1500
set interfaces xe-0/0/47 unit 10 vlan-id-list 1500
</pre>     
<p>
    The last thing we need to configure is the dot1q vlan (this needs to be configured without a tag):
</p>
<pre style="font-size:12px">
set vlans Q-in-Q interface xe-0/0/46.10
set vlans Q-in-Q interface xe-0/0/47.10
</pre>                  
<p>
    The complete configuration of the dot1q-tunnel on the QFX5100 will end up looking like this:
</p>
<pre style="font-size:12px">
set interfaces xe-0/0/46 description MX104-MARS_xe-1/2/0
set interfaces xe-0/0/46 vlan-tagging
set interfaces xe-0/0/46 encapsulation extended-vlan-bridge
set interfaces xe-0/0/46 unit 10 vlan-id-list 1500
set interfaces xe-0/0/46 unit 10 input-vlan-map push
set interfaces xe-0/0/46 unit 10 input-vlan-map vlan-id 10
set interfaces xe-0/0/46 unit 10 output-vlan-map pop

set interfaces xe-0/0/47 description MX104-JUPITER_xe-1/3/0
set interfaces xe-0/0/47 vlan-tagging
set interfaces xe-0/0/47 encapsulation extended-vlan-bridge
set interfaces xe-0/0/47 unit 10 vlan-id-list 1500
set interfaces xe-0/0/47 unit 10 input-vlan-map push
set interfaces xe-0/0/47 unit 10 input-vlan-map vlan-id 10
set interfaces xe-0/0/47 unit 10 output-vlan-map pop

set vlans Q-in-Q interface xe-0/0/46.10
set vlans Q-in-Q interface xe-0/0/47.10
</pre>                  
<p>
    Having applied the configuration, let’s move over to the MX-routers and configure a subinterface with vlan-id 1500.
</p>

<b>
    Mars:
</b>
<pre style="font-size:12px">
set interfaces xe-1/2/0 description QFX5100-xe-0/0/46
set interfaces xe-1/2/0 flexible-vlan-tagging
set interfaces xe-1/2/0 encapsulation flexible-ethernet-services
set interfaces xe-1/2/0 unit 1500 vlan-id 1500
set interfaces xe-1/2/0 unit 1500 family inet address 192.168.1.1/24
</pre>  
<b>
    Jupiter:
</b>                
<pre style="font-size:12px">
set interfaces xe-1/3/0 description QFX5100-xe-0/0/47
set interfaces xe-1/3/0 flexible-vlan-tagging
set interfaces xe-1/3/0 encapsulation flexible-ethernet-services
set interfaces xe-1/3/0 unit 1500 vlan-id 1500
set interfaces xe-1/3/0 unit 1500 family inet address 192.168.1.2/24
</pre>     
<p>
    After were done with the configuration and when we start sending traffic from the Mars router to the Jupiter router, the following will happen:
</p>                                                                

![QFX Q in Q](/img/qfx_qinq_2.png "QFX Q in Q")   

<p>
    The QFX will push vlan-id 10 on ingress and the tag is popped (removed) on egress. Let’s verify the connectivity between the routers:
</p>                
<pre style="font-size:12px">
play@MX104-TEST-HB:Mars> ping 192.168.1.2 size 1472 do-not-fragment
PING 192.168.1.2 (192.168.1.2): 1472 data bytes
1480 bytes from 192.168.1.2: icmp_seq=0 ttl=64 time=1.066 ms
1480 bytes from 192.168.1.2: icmp_seq=1 ttl=64 time=1.001 ms
^C
--- 192.168.1.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.001/1.034/1.066/0.032 ms

play@MX104-TEST-HB:Mars> show arp no-resolve | match 192.168.1.2
cc:e1:7f:7a:d4:a0 192.168.1.2     xe-1/2/0.1500        none
</pre>   
<p>
    On the QFX, we can see the following:
</p>
<pre style="font-size:12px">
play@QFX5100-play> show ethernet-switching table

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC, O - ovsdb MAC)


Ethernet switching table : 2 entries, 2 learned
Routing instance : default-switch
    Vlan                MAC                 MAC         Age    Logical
    name                address             flags              interface
    Q-in-Q              cc:e1:7f:7a:d3:f7   D             -   xe-0/0/46.10
    Q-in-Q              cc:e1:7f:7a:d4:a0   D             -   xe-0/0/47.10
</pre> 
<p>
    With the current configuration, the QFX doesn’t care how many tags the MX attaches to the packet. 
    To test this, let’s add vlan 1000 to the allowed vlan-list on the QFX and have the MX send packets with two tags across this link.
</p>

<b>
    Jupiter:
</b>                                  
<pre style="font-size:12px">
set interfaces xe-1/3/0 unit 1000 vlan-tags outer 1000
set interfaces xe-1/3/0 unit 1000 vlan-tags inner 2500
set interfaces xe-1/3/0 unit 1000 family inet address 10.0.0.1/24
</pre>     
<b>
    On Mars:
</b>                                  
<pre style="font-size:12px">
set interfaces xe-1/2/0 unit 1000 vlan-tags outer 1000
set interfaces xe-1/2/0 unit 1000 vlan-tags inner 2500
set interfaces xe-1/2/0 unit 1000 family inet address 10.0.0.2/24
</pre> 
<b>
    On the QFX:
</b>                                  
<pre style="font-size:12px">
set interfaces xe-0/0/46 unit 10 vlan-id-list 1000
set interfaces xe-0/0/47 unit 10 vlan-id-list 1000
</pre>     
<p>
    This last command will only add the vlan to the vlan-id-list, it will not remove the previously configured vlan:
</p>
<pre style="font-size:12px">
play@QFX5100-play> show configuration interfaces xe-0/0/47
description MX104-JUPITER_xe-1/3/0;
vlan-tagging;
encapsulation extended-vlan-bridge;
unit 10 {
    vlan-id-list [ 1000 1500 ];
    input-vlan-map {
        push;
        vlan-id 10;
    }
    output-vlan-map pop;
}
</pre>     
<p>
    After applying all the configuration, we can observe the following:
</p>                
<pre style="font-size:12px">
play@MX104-TEST-HB:Jupiter> show interfaces xe-1/3/0.1000
  Logical interface xe-1/3/0.1000 (Index 462) (SNMP ifIndex 656)
    Flags: SNMP-Traps 0x0 VLAN-Tag [ 0x8100.1000 0x8100.2500 ]  Encapsulation: ENET2
    Input packets : 24
    Output packets: 35
    Protocol inet, MTU: 1500
      Flags: Sendbcast-pkt-to-re
      Addresses, Flags: Is-Preferred Is-Primary
        Destination: 10.0.0/24, Local: 10.0.0.1, Broadcast: 10.0.0.255
    Protocol multiservice, MTU: Unlimited

play@MX104-TEST-HB:Jupiter> ping 10.0.0.2
PING 10.0.0.2 (10.0.0.2): 56 data bytes
64 bytes from 10.0.0.2: icmp_seq=0 ttl=64 time=0.643 ms
64 bytes from 10.0.0.2: icmp_seq=1 ttl=64 time=0.593 ms
^C
--- 10.0.0.2 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.593/0.618/0.643/0.025 ms
</pre>                   
<p>
    The ‘old’ EX way was a little more straightforward. On an EX switch (most of them) you can simply configure this:
</p>
<pre style="font-size:12px">
set ethernet-switching-options dot1q-tunneling ether-type 0x8100
set vlans example vlan-id 2000
set vlans example dot1q-tunneling customer-vlans 1-4094
</pre>     
<p>
    And apply it to the interface:
</p>
<pre style="font-size:12px">
set interfaces ge-0/1/3 unit 0 family ethernet-switching port-mode trunk
set interfaces ge-0/1/3 unit 0 family ethernet-switching vlan members 2000
</pre>                 
<p>
    And if you want to tunnel (v)stp, lldp, etc, just add the following:
</p>
<pre style="font-size:12px">
set vlans example dot1q-tunneling layer2-protocol-tunneling all
</pre>                 
<p>
    The knob to tunnel layer 2 protocols on the QFX can be found in the [protocols layer2-control] configuration stanza. 
    This doesn not seem work with the QFX configuration I presented here.
    It does work on interfaces with the etherswitching family enabled. 
    Unfortunately I have not been able to get dot1q-tunneling working with the etherswitching family enabled on an interface.
</p>
