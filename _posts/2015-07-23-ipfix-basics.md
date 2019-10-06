---
layout: post
title: IPFIX configuration example
tags: [juniper]
image: /img/juniper_logo.jpg
---
    
<p>
    Turning on <b>IPFIX</b> (IP Flow Information Export) on Juniper MX is a good idea if you want to know what’s going on.
    Not only can it provide you with a tremendous insight into the traffic traversing your network, you can also use the information provided by IPFIX to automatically divert traffic or thwart a DDOS.                    
</p>
<p>
    IPFIX is an IETF documented standard that describes how to format IP flow information. 
    The information inside a flow contains the source and destination IP/port, the protocol, ToS and ingress interface.
    
</p>


![IPFIX](/img/ipfix-1.png "IPFIX") 

<p>
    After configuring all the Internet connected interfaces for IPFIX, the networking device starts to collect flows.
    These flows can be send to your flow collector. The information inside these flows will reveal a lot of what is going on. You can look at this information (near) real-time or you can make the collector generate periodic reports based on any the information gathered by IPFIX. 
</p>
<p>
    The basic IPFIX configuration on an MX router is very easy. You do have to go through 4 different configuration stanza’s to get things going.  I’ll demonstrate the configuration using the following setup:
</p>
        
![IPFIX](/img/ipfix-2.png "IPFIX") 
   
<pre style="font-size:12px">
set services flow-monitoring version-ipfix template ipv4 ipv4-template
set services flow-monitoring version-ipfix template ipv6 ipv6-template

set forwarding-options sampling instance inline input rate 50
set forwarding-options sampling instance inline family inet output flow-server 10.0.0.1 port 2055
set forwarding-options sampling instance inline family inet output flow-server 10.0.0.1 version-ipfix template ipv4
set forwarding-options sampling instance inline family inet output inline-jflow source-address 1.1.1.1
set forwarding-options sampling instance inline family inet6 output flow-server 10.0.0.1 port 2055
set forwarding-options sampling instance inline family inet6 output flow-server 10.0.0.1 version-ipfix template ipv6
set forwarding-options sampling instance inline family inet6 output inline-jflow source-address 1.1.1.1

set chassis fpc 3 sampling-instance inline
set chassis fpc 3 inline-services flow-table-size ipv4-flow-table-size 5
set chassis fpc 3 inline-services flow-table-size ipv6-flow-table-size 5

set interfaces xe-3/2/0 unit 0  family inet sampling input
set interfaces xe-3/2/0 unit 0  family inet6 sampling input
set interfaces xe-3/2/1 unit 0  family inet sampling input
set interfaces xe-3/2/1 unit 0  family inet6 sampling input
</pre>                
<p>
    To see if the MX is gathering and exporting flows, you can issue the following command:
</p>
<pre style="font-size:12px">
play@MX480> show services accounting flow inline-jflow fpc-slot 3
  Flow information
    FPC Slot: 3
    Flow Packets: 672557996, Flow Bytes: 395793951925
    Active Flows: 1763, Total Flows: 239316095
    Flows Exported: 224677847, Flow Packets Exported: 94025957
    Flows Inactive Timed Out: 209386360, Flows Active Timed Out: 29927972

    IPv4 Flows:
    IPv4 Flow Packets: 672273557, IPv4 Flow Bytes: 395740955664
    IPv4 Active Flows: 1761, IPv4 Total Flows: 239042164
    IPv4 Flows Exported: 224405780, IPv4 Flow Packets exported: 93754406
    IPv4 Flows Inactive Timed Out: 209114350, IPv4 Flows Active Timed Out: 29926053

    IPv6 Flows:
    IPv6 Flow Packets: 284439, IPv6 Flow Bytes: 52996261
    IPv6 Active Flows: 2, IPv6 Total Flows: 273931
    IPv6 Flows Exported: 272067, IPv6 Flow Packets Exported: 271551
    IPv6 Flows Inactive Timed Out: 272010, IPv6 Flows Active Timed Out: 1919</pre>
<p>
FYIs:
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; tested on MPC2, 12.3R8.7 ( MPC2 can export about 200K flows/second, MPC3 400k)
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; will not kill your RE
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; will reboot your FPC or TFEB when you configure or alter the flow-table size
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; will not work on DPCE line-cards
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; MX80 and MX104 configuration is slightly <a href="http://www.juniper.net/techpubs/en_US/junos14.2/topics/task/configuration/inline-flow-monitoring-mx80.html">different</a>, you have to reference the tfeb in the chassis stanza
</p>
                