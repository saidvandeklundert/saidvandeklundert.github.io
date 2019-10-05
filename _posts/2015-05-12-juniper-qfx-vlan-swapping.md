---
layout: post
title: Juniper QFX vlan-swapping
image: /img/juniper_logo.jpg
---
   
<p>
    This is a quick and short article on how to perform <b>vlan-swapping on a Juniper QFX5100</b>. 
    I was used to tunneling vlans in a QFX5100 by using the push-operation available through a vlan-map. With this in mind I was struggling to get vlan translation on the QFX5100 working. I was trying to accomplish vlan-swapping through the same type of configuration.
</p>
<p>
    Turns out swapping vlan-id’s is a lot easier. In the following lab setup, I will make the QFX translate vlans between two of the MX104’s interfaces:
</p>            
            

![VLAN swapping](/img/qfx-vlan-swapping-1.png "VLAN swapping") 

<p>
    On the MX104, I will configure a routing-instance ‘QFX-VLAN-<font color='red'>501</font>’ which will be tied to interface xe-1/2/0.501. 
    This interface will be configured with vlan vlan-id 501. I will also configure a routing-instance called ‘QFX-VLAN-<font color='red'>500</font>’. 
    Interface xe-1/3/0.500, with vlan-id 500, will be placed inside this routing instance. This is configured in the following way:
</p>
               

![VLAN swapping](/img/qfx-vlan-swapping-2.png "VLAN swapping") 

  
<p>
    Now for the QFX part. On both of the QFX interfaces, the ‘extended-vlan-bridge’ encapsulation will need to be enabled. Besides the encapsulation, a logical interface with the desired vlan-id is also required. This can be configured like this;
</p>
                

![VLAN swapping](/img/qfx-vlan-swapping-3.png "VLAN swapping") 

    
<p>
    At the bottom of the picture, I listed the vlan configuration that was applied to the QFX.  Notice that vlan-id 501 was configured for interface xe-0/0/46 unit 501 in the interface configuration stanza. However, in the vlan configuration stanza, the interface itself was placed in vlan 500.
</p>
<p>
    With this configuration, the QFX switch will (automagically) start swapping vlan-id’s in and out of interface xe-0/0/46. 
</p>
<p>
    Let’s examine both of the QFX interfaces with a simple show command;
</p>                
<pre style="font-size:12px">
play@Aether> show interfaces xe-0/0/46.501
  Logical interface xe-0/0/46.501 (Index 566) (SNMP ifIndex 525)
    Flags: SNMP-Traps 0x20004000 VLAN-Tag [ 0x8100.501 ] <font color='red'>In(swap .500) Out(swap .501)</font>  Encapsulation: Extended-VLAN-Bridge
    Input packets : 0
    Output packets: 0
    Protocol eth-switch, MTU: 1518

play@Aether> show interfaces xe-0/0/47.500
  Logical interface xe-0/0/47.500 (Index 569) (SNMP ifIndex 523)
    Flags: SNMP-Traps 0x20004000 VLAN-Tag [ 0x8100.500 ]  Encapsulation: Extended-VLAN-Bridge
    Input packets : 0
    Output packets: 0
    Protocol eth-switch, MTU: 1518                  
</pre>
<p>
    Interface xe-0/0/47 is not showing anything out of the ordinary. This is because nothing extra is happening on that interface. Interface xe-0/0/46 is showing the vlan-operation. Incoming frames have their vlan-id swapped to 500 and outgoing frames have their vlan-id swapped to 501.
</p>
<p>
    Let’s verify this by sending some packets from the MX104 ‘QFX-VLAN-501’ instance to the ‘QFX-VLAN-500’ instance like this:
</p>
<pre style="font-size:12px">
play@MX104-TEST-HB> ping routing-instance QFX-VLAN-501 source 1.1.1.1 1.1.1.2 count 5
PING 1.1.1.2 (1.1.1.2): 56 data bytes
64 bytes from 1.1.1.2: icmp_seq=0 ttl=64 time=0.687 ms
64 bytes from 1.1.1.2: icmp_seq=1 ttl=64 time=0.601 ms
64 bytes from 1.1.1.2: icmp_seq=2 ttl=64 time=0.648 ms
64 bytes from 1.1.1.2: icmp_seq=3 ttl=64 time=0.625 ms
64 bytes from 1.1.1.2: icmp_seq=4 ttl=64 time=0.619 ms

--- 1.1.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 0.601/0.636/0.687/0.030 ms

play@MX104-TEST-HB>                    
</pre>
<p>
    So there is connectivity between the two interfaces. What just happened was the following:
</p>

![VLAN swapping](/img/qfx-vlan-swapping-4.png "VLAN swapping") 

    
<p>
    Packets are send from the xe-1/2/0.501 interface of the MX104. These packets, tagged with vlan-id 501, are received on the QFX xe-0/0/46.501 interface. Upon the receipt of these packets, the QFX will rewrite the vlan-tag to 500 and perform all relevant lookups in the MAC table of vlan 500. After this lookup, the QFX will forward this packet out interface xe-0/0/47.500. The packet will arrive on the xe-1/3/0.500 interface with the correct vlan-id.
</p>
<p>
    When consulting the MAC table, be sure to insert the vlan-id that the vlan was configured with. So in our case, vlan-id 500 will contain the mac-address of both the MX104’s interfaces. Vlan-id 501 will not contain any entries;
</p>
<pre style="font-size:12px">
play@Aether> show ethernet-switching table vlan-id 500

MAC flags (S - static MAC, D - dynamic MAC, L - locally learned, P - Persistent static
           SE - statistics enabled, NM - non configured MAC, R - remote PE MAC)


Ethernet switching table : 2 entries, 2 learned
Routing instance : default-switch
    Vlan                MAC                 MAC         Age    Logical
    name                address             flags              interface
    VLAN-500-501        64:64:9b:dd:60:b7   D             -   xe-0/0/46.501
    VLAN-500-501        64:64:9b:dd:61:60   D             -   xe-0/0/47.500

{master:0}
play@Aether> show ethernet-switching table vlan-id 501

{master:0}
play@Aether>                    
</pre>
<p>
    I was able to perform this on a QFX5100 running 13.2X51-D30.4. On this software level, it is possible to swap vlan-id’s between two interfaces that are configured with the ‘extended-vlan-bridge’ encapsulation. Hopefully, somewhere in future releases, Juniper will make it so that two interfaces with different encapsulations will also allow for vlan-swapping.
</p>
