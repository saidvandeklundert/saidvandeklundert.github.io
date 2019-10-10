---
layout: post
title: Setting up your own vSRX lab
tags: [juniper]
image: /img/juniper_logo.jpg
---


<p>
    Recently, I’ve been having some fun with the <b>vSRX</b>.
    I wanted to share the lab I created so others can see how easy it is to get things going with the vSRX. The vSRX doesn’t require a lot of resources and it is a really nice way to get acquainted with both the routing and firewalling capabilities that Junos has to offer. Labbing with a vSRX allows you to quickly test and play with routing-protocols (OSPF(v3), IS-IS, BGP), stateful firewalling, NAT, IPsec or even MPLS. This guide offers an insight into how you can setup a lab and it will provide you with some basic configurations to get started yourself.                  
</p>
<p>
    I used several x3550 hosts, running ESXi, connected to a switch (EX4200):
</p>

![vSRX](/img/vsrx_lab_physical_setup.png "vSRX") 

<p>
    The lab contains two firewalls and two routers. Each vSRX runs OSPF3 to distribute prefix information for both IPv4 and IPv6:
</p>

![vSRX](/img/vsrx_lab_logical_setup.png "vSRX") 

<p>
    The vSRX-01 and vSRX-02 are used for firewalling. In between the firewalls, there are the vSRX-router-01 and the vSRX-router-02. 
    These have the flow forwarding mode set to ‘<b>packet based</b>’ instead of ‘<b>flow based</b>’ (basically turning them into routers).
</p>
<p>
    On every vSRX, the first vNIC (ge-0/0/0, not in the picture) was placed into an untagged port-group and it was configured with a management IP:
</p>  

![vSRX](/img/vsrx_lab_mgmt.png "vSRX") 

<p>
    All other vSRX interfaces (ge-0/0/1 and ge-0/0/2) were placed into a port-group that was tagged:
</p>  

![vSRX](/img/vsrx_lab_vmware_trunk.png "vSRX") 

<p>
   To deploy the SRX in VMware, go to File > deploy OVF Template > next > next > accept > next > name the VM > next > select cluster > select resource pool > select storage > thin provision it > next (add vNICs and networks later) and finish.
</p>
<p>
    After the VM is created, edit the VM. Add two vNICs so you’ll have a total of three interfaces. Put the first vNIC in the management port-group and place the rest into the tagged port-group. Press start and wait for Junos.
</p>
<p>
    When you boot the vSRX in Vmware for the first time, you can only access it via the console screen. To be able to quickly copy-paste the configuration into the vSRX, the vSRX will need to have an IP address and it needs to be configured for telnet/ssh access. To avoid too much typing, simply make 1 vSRX managable like this: 
</p>

<pre style="font-size:12px">
root
cli
configure
delete
set system host-name <font color='red'>vSRX</font>
set system root-authentication plain-text-password
playplay1
set system login user play uid 9999
set system login user play class super-user
set system login user play authentication plain-text-password
playplay1
set system services telnet
set interfaces ge-0/0/0 unit 0 family inet address <font color='red'>10.10.13.200/24</font>
set security zones functional-zone management interfaces ge-0/0/0.0 host-inbound-traffic system-services telnet
commit and-quit
</pre> 
<p>
    Power down this first vSRX and clone it 3 times. Then boot the VMs one by one and copy paste the configuration below into the vSRX:
    
</p>
                
                
<b>vSRX-01</b>:

<pre style="font-size:12px">
delete
set system host-name vSRX-01
set system root-authentication encrypted-password "$1$brjswqXa$/5315/zMuQuIiUu8kQT8U/"
set system login user play uid 9999
set system login user play class super-user
set system login user play authentication encrypted-password "$1$Z/K/ZEGJ$ydYmwT99wlfVJXgy8ZBhn."
set system services ssh
set system services telnet
set system syslog user * any emergency
set system syslog file messages any any
set system syslog file messages authorization info
set system syslog file interactive-commands interactive-commands any
set interfaces ge-0/0/0 description mgmt
set interfaces ge-0/0/0 unit 0 family inet address 10.10.13.201/24
set interfaces ge-0/0/1 vlan-tagging
set interfaces ge-0/0/1 unit 3009 description vSRX-router-01
set interfaces ge-0/0/1 unit 3009 vlan-id 3009
set interfaces ge-0/0/1 unit 3009 family inet address 192.168.50.2/30
set interfaces ge-0/0/1 unit 3009 family inet6 address 2001:db8::0/127
set interfaces ge-0/0/2 vlan-tagging
set interfaces ge-0/0/2 unit 3001 description VM
set interfaces ge-0/0/2 unit 3001 vlan-id 3001
set interfaces ge-0/0/2 unit 3001 family inet address 172.16.0.1/24
set interfaces ge-0/0/2 unit 3001 family inet6 address 2001:db8:1::1/64
set interfaces lo0 unit 0 family inet address 192.168.1.1/32 primary
set interfaces lo0 unit 0 family inet6
set routing-options router-id 192.168.1.1
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/1.3009 interface-type p2p
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.3001 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/1.3009 interface-type p2p
set protocols ospf3 area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.3001 passive
set security forwarding-options family inet6 mode flow-based
set security zones functional-zone management interfaces ge-0/0/0.0 host-inbound-traffic system-services telnet
set security zones functional-zone management interfaces ge-0/0/0.0 host-inbound-traffic system-services ssh
set security zones security-zone wan interfaces ge-0/0/1.3009 host-inbound-traffic system-services ping
set security zones security-zone wan interfaces ge-0/0/1.3009 host-inbound-traffic protocols ospf3
set security zones security-zone wan interfaces lo0.0 host-inbound-traffic system-services ping
set security zones security-zone lan interfaces ge-0/0/2.3001 host-inbound-traffic system-services ping
set security zones security-zone lan interfaces ge-0/0/2.3001 host-inbound-traffic system-services dhcp
set routing-instances management instance-type virtual-router
set routing-instances management interface ge-0/0/0.0
set routing-instances management routing-options static route 0.0.0.0/0 next-hop 10.10.13.1
set applications application junos-telnet inactivity-timeout never
set applications application junos-ssh inactivity-timeout never
</pre>

<b>vSRX-02</b>:

<pre style="font-size:12px">
delete
set system host-name vSRX-02
set system root-authentication encrypted-password "$1$brjswqXa$/5315/zMuQuIiUu8kQT8U/"
set system login user play uid 9999
set system login user play class super-user
set system login user play authentication encrypted-password "$1$Z/K/ZEGJ$ydYmwT99wlfVJXgy8ZBhn."
set system services ssh
set system services telnet
set system syslog user * any emergency
set system syslog file messages any any
set system syslog file messages authorization info
set system syslog file interactive-commands interactive-commands any
set interfaces ge-0/0/0 description mgmt
set interfaces ge-0/0/0 unit 0 family inet address 10.10.13.202/24
set interfaces ge-0/0/1 vlan-tagging
set interfaces ge-0/0/1 unit 3011 description vSRX-router-02
set interfaces ge-0/0/1 unit 3011 vlan-id 3011
set interfaces ge-0/0/1 unit 3011 family inet address 192.168.50.6/30
set interfaces ge-0/0/1 unit 3011 family inet6 address 2001:db8::5/127
set interfaces ge-0/0/2 vlan-tagging
set interfaces ge-0/0/2 unit 3002 description VM
set interfaces ge-0/0/2 unit 3002 vlan-id 3002
set interfaces ge-0/0/2 unit 3002 family inet address 172.16.1.1/24
set interfaces ge-0/0/2 unit 3002 family inet6
set interfaces lo0 unit 0 family inet address 192.168.1.2/32
set interfaces lo0 unit 0 family inet6
set routing-options router-id 192.168.1.2
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/1.3011 interface-type p2p
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.3002 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/1.3011 interface-type p2p
set protocols ospf3 area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.3002 passive
set security forwarding-options family inet6 mode flow-based
set security policies from-zone wan to-zone wan policy 1 description allow-ping-to-loopback
set security policies from-zone wan to-zone wan policy 1 match source-address any-ipv4
set security policies from-zone wan to-zone wan policy 1 match destination-address loopback
set security policies from-zone wan to-zone wan policy 1 match application junos-ping
set security policies from-zone wan to-zone wan policy 1 then permit
set security zones functional-zone management interfaces ge-0/0/0.0 host-inbound-traffic system-services telnet
set security zones functional-zone management interfaces ge-0/0/0.0 host-inbound-traffic system-services ssh
set security zones security-zone wan address-book address loopback 192.168.1.2/32
set security zones security-zone wan interfaces ge-0/0/1.3011 host-inbound-traffic system-services ping
set security zones security-zone wan interfaces ge-0/0/1.3011 host-inbound-traffic protocols ospf3
set security zones security-zone wan interfaces lo0.0 host-inbound-traffic system-services ping
set security zones security-zone lan interfaces ge-0/0/2.3002
set routing-instances management instance-type virtual-router
set routing-instances management interface ge-0/0/0.0
set routing-instances management routing-options static route 0.0.0.0/0 next-hop 10.10.13.1
set applications application junos-telnet inactivity-timeout never
set applications application junos-ssh inactivity-timeout never
</pre>


<b>vSRX-router-01</b>:

<pre style="font-size:12px">
delete
set system host-name vSRX-router-01
set system root-authentication encrypted-password "$1$brjswqXa$/5315/zMuQuIiUu8kQT8U/"
set system login user play uid 9999
set system login user play class super-user
set system login user play authentication encrypted-password "$1$Z/K/ZEGJ$ydYmwT99wlfVJXgy8ZBhn."
set system services ssh
set system services telnet
set system syslog user * any emergency
set system syslog file messages any any
set system syslog file messages authorization info
set system syslog file interactive-commands interactive-commands any
set interfaces ge-0/0/0 description mgmt
set interfaces ge-0/0/0 unit 0 family inet address 10.10.13.203/24
set interfaces ge-0/0/1 vlan-tagging
set interfaces ge-0/0/1 unit 3009 description vSRX-01
set interfaces ge-0/0/1 unit 3009 vlan-id 3009
set interfaces ge-0/0/1 unit 3009 family inet address 192.168.50.1/30
set interfaces ge-0/0/1 unit 3009 family inet6 address 2001:db8::1/127
set interfaces ge-0/0/2 vlan-tagging
set interfaces ge-0/0/2 unit 3010 description vSRX-router-02
set interfaces ge-0/0/2 unit 3010 vlan-id 3010
set interfaces ge-0/0/2 unit 3010 family inet address 10.1.0.1/30
set interfaces ge-0/0/2 unit 3010 family inet6 address 2001:db8::2/127
set interfaces lo0 unit 0 family inet address 10.0.0.1/32
set interfaces lo0 unit 0 family inet6
set routing-options router-id 10.0.0.1
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.3010 interface-type p2p
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/1.3009 interface-type p2p
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.3010 interface-type p2p
set protocols ospf3 area 0.0.0.0 interface ge-0/0/1.3009 interface-type p2p
set protocols ospf3 area 0.0.0.0 interface lo0.0 passive
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
set routing-instances management instance-type virtual-router
set routing-instances management interface ge-0/0/0.0
set routing-instances management routing-options static route 0.0.0.0/0 next-hop 10.10.13.1
</pre>

<b>vSRX-router-02</b>:

<pre style="font-size:12px">
delete
set system host-name vSRX-router-02
set system root-authentication encrypted-password "$1$brjswqXa$/5315/zMuQuIiUu8kQT8U/"
set system login user play uid 9999
set system login user play class super-user
set system login user play authentication encrypted-password "$1$Z/K/ZEGJ$ydYmwT99wlfVJXgy8ZBhn."
set system services ssh
set system services telnet
set system syslog user * any emergency
set system syslog file messages any any
set system syslog file messages authorization info
set system syslog file interactive-commands interactive-commands any
set interfaces ge-0/0/0 description mgmt
set interfaces ge-0/0/0 unit 0 family inet address 10.10.13.204/24
set interfaces ge-0/0/1 vlan-tagging
set interfaces ge-0/0/1 unit 3011 description vSRX-02
set interfaces ge-0/0/1 unit 3011 vlan-id 3011
set interfaces ge-0/0/1 unit 3011 family inet address 192.168.50.5/30
set interfaces ge-0/0/1 unit 3011 family inet6 address 2001:db8::4/127
set interfaces ge-0/0/2 vlan-tagging
set interfaces ge-0/0/2 unit 3010 description vSRX-router-01
set interfaces ge-0/0/2 unit 3010 vlan-id 3010
set interfaces ge-0/0/2 unit 3010 family inet address 10.1.0.2/30
set interfaces ge-0/0/2 unit 3010 family inet6 address 2001:db8::3/127
set interfaces ge-0/0/2 unit 3010 family inet6 address 2001:db8:2::1/64
set interfaces lo0 unit 0 family inet address 10.0.0.2/32
set interfaces lo0 unit 0 family inet6
set routing-options router-id 10.0.0.2
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.3010 interface-type p2p
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/1.3011 interface-type p2p
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface lo0.0 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/1.3011 interface-type p2p
set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.3010 interface-type p2p
set protocols ospf3 area 0.0.0.0 interface lo0.0 passive
set security forwarding-options family inet6 mode packet-based
set security forwarding-options family mpls mode packet-based
set routing-instances management instance-type virtual-router
set routing-instances management interface ge-0/0/0.0
set routing-instances management routing-options static route 0.0.0.0/0 next-hop 10.10.13.1
</pre>

<p>
    After a commit and-quit on every vSRX, reset all of them and check the following:
</p>
<pre style="font-size:12px">
play@<font color='red'>vSRX-01</font>> ping source 192.168.1.1 192.168.1.2 rapid
PING 192.168.1.2 (192.168.1.2): 56 data bytes
!!!!!
--- 192.168.1.2 ping statistics ---
5 packets transmitted, 5 packets received, 0% packet loss
round-trip min/avg/max/stddev = 15.037/18.973/20.195/1.974 ms

play@<font color='red'>vSRX-02</font>> show security flow session destination-prefix 192.168.1.2
Session ID: 49816, Policy name: 1/4, Timeout: 2, Valid
  In: 192.168.1.1/0 --> 192.168.1.2/39187;icmp, If: ge-0/0/1.3011, Pkts: 1, Bytes: 84
  Out: 192.168.1.2/39187 --> 192.168.1.1/0;icmp, If: .local..0, Pkts: 1, Bytes: 84
</pre>  

                
<p>
    This lab is nothing really groundbreaking, but it offers a nice start to get into the vSRX. 
    It is easy to expand the lab start playing with different scenarios. A last example where I make it possible for two VM's to ping each other:
</p>
<b>vSRX-01:</b>
<pre style="font-size:12px">
set interfaces ge-0/0/2 unit 3001 description VM
set interfaces ge-0/0/2 unit 3001 vlan-id 3001
set interfaces ge-0/0/2 unit 3001 family inet address 172.16.0.1/24
set interfaces ge-0/0/2 unit 3001 family inet6

set security zones security-zone lan address-book address lan-hosts range-address 172.16.1.2 to 172.16.1.45

set system services dhcp pool 172.16.0.0/24 address-range low 172.16.0.2
set system services dhcp pool 172.16.0.0/24 address-range high 172.16.0.45
set system services dhcp pool 172.16.0.0/24 domain-name play.play
set system services dhcp pool 172.16.0.0/24 name-server 8.8.4.4
set system services dhcp pool 172.16.0.0/24 router 172.16.0.1
set system services dhcp pool 172.16.0.0/24 default-lease-time 36000

set security zones security-zone lan interfaces ge-0/0/2.3001 host-inbound-traffic system-services dhcp
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.3001 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.3001 passive
set security zones security-zone lan interfaces ge-0/0/2.3001 host-inbound-traffic system-services ping
set security zones security-zone lan interfaces ge-0/0/2.3001 host-inbound-traffic system-services dhcp

set security policies from-zone wan to-zone lan policy 1 description allow-ping-all-to-dhcp-range
set security policies from-zone wan to-zone lan policy 1 match source-address any-ipv4
set security policies from-zone wan to-zone lan policy 1 match destination-address lan-hosts
set security policies from-zone wan to-zone lan policy 1 match application junos-ping
set security policies from-zone wan to-zone lan policy 1 then permit

set security policies from-zone lan to-zone wan policy 1 description allow-ping
set security policies from-zone lan to-zone wan policy 1 match source-address lan-hosts
set security policies from-zone lan to-zone wan policy 1 match destination-address any-ipv4
set security policies from-zone lan to-zone wan policy 1 match application junos-ping
set security policies from-zone lan to-zone wan policy 1 then permit
</pre>                
<b>vSRX-02:</b>
<pre style="font-size:12px">
set interfaces ge-0/0/2 unit 3002 description VM
set interfaces ge-0/0/2 unit 3002 vlan-id 3002
set interfaces ge-0/0/2 unit 3002 family inet address 172.16.1.1/24
set interfaces ge-0/0/2 unit 3002 family inet6

set security zones security-zone lan address-book address lan-hosts range-address 172.16.1.2 to 172.16.1.45

set system services dhcp pool 172.16.1.0/24 address-range low 172.16.1.2
set system services dhcp pool 172.16.1.0/24 address-range high 172.16.1.45
set system services dhcp pool 172.16.1.0/24 domain-name play.play
set system services dhcp pool 172.16.1.0/24 name-server 8.8.4.4
set system services dhcp pool 172.16.1.0/24 router 172.16.1.1
set system services dhcp pool 172.16.1.0/24 default-lease-time 36000

set security zones security-zone lan interfaces ge-0/0/2.3002 host-inbound-traffic system-services dhcp
set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface ge-0/0/2.3002 passive
set protocols ospf3 area 0.0.0.0 interface ge-0/0/2.3002 passive
set security zones security-zone lan interfaces ge-0/0/2.3002 host-inbound-traffic system-services ping
set security zones security-zone lan interfaces ge-0/0/2.3002 host-inbound-traffic system-services dhcp

set security policies from-zone wan to-zone lan policy 1 description allow-ping-all-to-dhcp-range
set security policies from-zone wan to-zone lan policy 1 match source-address any-ipv4
set security policies from-zone wan to-zone lan policy 1 match destination-address lan-hosts
set security policies from-zone wan to-zone lan policy 1 match application junos-ping
set security policies from-zone wan to-zone lan policy 1 then permit

set security policies from-zone lan to-zone wan policy 1 description allow-ping
set security policies from-zone lan to-zone wan policy 1 match source-address lan-hosts
set security policies from-zone lan to-zone wan policy 1 match destination-address any-ipv4
set security policies from-zone lan to-zone wan policy 1 match application junos-ping
set security policies from-zone lan to-zone wan policy 1 then permit
</pre>
                
                
<p>
                    
After applying this configuration, I started two VMs and I put them into an untagged port-group that matched the vlan-id I used configuring the subinterfaces of the SRX. After sending some traffic from the VM located behind vSRX-01 to the VM located on vSRX-02, I was able to observe the following:
</p>

![vSRX](/img/vsrx_lab_vm_to_vm.png "vSRX") 

<pre style="font-size:12px">
play@vSRX-02> show security flow session destination-prefix 172.16.1.10
Session ID: 4617, Policy name: 1/5, Timeout: 2, Valid
  <font color='red'>In</font>: <font color='red'>172.16.0.10</font>/42127 --> <font color='red'>172.16.1.10</font>/1;icmp, If: <font color='red'>ge-0/0/1.3011</font>, Pkts: 1, Bytes: 60
  <font color='red'>Out</font>: <font color='red'>172.16.1.10</font>/1 --> <font color='red'>172.16.0.10</font>/42127;icmp, If: <font color='red'>ge-0/0/2.3002</font>, Pkts: 1, Bytes: 60
</pre> 
                
<p>
    Some verification commands I used:
</p>
<pre style="font-size:12px">
show route terse
show ospf3 neighbor
show system services dhcp pool
show system services dhcp binding
show security flow session
show security flow status
show security flow session summary
</pre> 
                
<p>
    I think the vSRX is more than worth your while. First of all, the vSRX runs Junos. This offers a robust and featurific platform. Besides this, the performance offered by the vSRX makes it suitable for a lot of different scenarios that do not require too much IPsec (figures came from the vSRX datasheet):
</p>

![vSRX performance](/img/vsrx_performance.png "vSRX performance") 

<p>
    
    To really get into the SRX, I can recommend ‘Juniper SRX Series’ or the older 'Junos Security' from the O'Reilly Media. 
    Other than those books, just check the Juniper website for the evaluation download <a href="http://www.juniper.net/us/en/dm/free-vsrx-trial/">here</a> and start labbing yourself.                    
</p>