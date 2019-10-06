---
layout: post
title: Juniper OSPFv3 IPsec authentication
tags: [juniper]
image: /img/juniper_logo.jpg
---


<p>
Though the <b>OSPFv3</b> protocol does not offer a built-in authentication method, <b>IPsec</b> can be used to secure protocol exchanges between devices running OSPFv3.                      
To authenticate OSPFv3 on a Juniper device, you first start out with the configuration of a <b>Security Association</b> (<b>SA</b>). 
The SA describes how the devices will communicate securely by defining the algorithms and keys that will be used.
</p>

<p>
The SA can be configured in the following way:
</p>
                
<pre style="font-size:12px">
set security ipsec security-association ospf3-sa mode transport
set security ipsec security-association ospf3-sa manual direction bidirectional protocol esp
set security ipsec security-association ospf3-sa manual direction bidirectional spi 256
set security ipsec security-association ospf3-sa manual direction bidirectional authentication algorithm hmac-sha1-96
set security ipsec security-association ospf3-sa manual direction bidirectional authentication key ascii-text 01234567890123456789
</pre>                
                
<p>
After the definition of this SA, you simply reference it after the OSPFv3 interface configuration. The following is an example configuration for a Juniper MX router:
</p>

<pre style="font-size:12px">
set protocols ospf3 area 0.0.0.100 interface ge-0/0/2.3008 ipsec-sa ospf3-sa
</pre> 

<p>
Securing  virtual-link is done in a similar way:
</p>
<pre style="font-size:12px">
set protocols ospf3 area 0.0.0.0 virtual-link neighbor-id 192.168.0.4 transit-area 0.0.0.100 ipsec-sa ospf3-sa
</pre>    

<p>
IPsec SA settings can be verified using the the ‘<b>show ipsec security-associations detail</b>’ command.
To verify what IPsec-SA is in use for OSPFv3 between a given set of Juniper devices, you can issue the following command;
</p>

<pre style="font-size:12px">
play@MX480> show ospf3 interface detail
ge-0/0/2.3008       PtToPt  0.0.0.100       0.0.0.0         0.0.0.0            1
  Address fe80::250:560b:c097:4df1, Prefix-length 64
  OSPF3-Intf-index 3, Type P2P, MTU 1500, Cost 10
  Adj count: 1, Router LSA ID: 0
  Hello 10, Dead 40, ReXmit 5, Not Stub
  <font color='red'>IPSec SA</font> name: <font color='red'>ospf3-sa</font>
  Protection type: None
  </pre> 

<p>
This method of securing protocol information can also be used for OSPFv2. 
I initially thought that the authentication options for OSPFv2 were ‘none’, ‘plain-text’ or ‘MD5’. Same as with OSPFv3, the SA can be used for OSPFv2 simply by referencing it in the OSPF configuration stanza like this:
</p>

<pre style="font-size:12px">
set protocols ospf area 0.0.0.0 interface ge-0/0/1.3001 ipsec-sa ospf3-sa
</pre>

<p>
And for verifications, you can use the same command as with OSPFv3:
</p>

<pre style="font-size:12px">
play@MX480> show ospf interface extensive
Interface           State   Area            DR ID           BDR ID          Nbrs
ge-0/0/1.3001       DR      0.0.0.0         192.168.0.5     192.168.0.4        1
  Type: LAN, Address: 192.168.50.2, Mask: 255.255.255.252, MTU: 1500, Cost: 1
  DR addr: 192.168.50.2, BDR addr: 192.168.50.1, Priority: 128
  Adj count: 1
  Hello: 10, Dead: 40, ReXmit: 5, Not Stub
  <font color='red'>IPSec SA</font> name: <font color='red'>ospf3-sa</font>
  Auth type: None
  Protection type: None
  Topology default (ID 0) -> Cost: 1
</pre>  

<p>
Keep in mind that if you are using OSPFv3 to distribute IPv4 prefix information, interfaces under the ipv4-unicast realm need to be configured to communicate securely as well:
</p>

<pre style="font-size:12px">set protocols ospf3 realm ipv4-unicast area 0.0.0.0 interface xe-0/3/0.1503 ipsec-sa ospf3-sa</pre>  

<p>
And to verify this:
</p>

<pre style="font-size:12px">play@MX480> show ospf3 interface realm ipv4-unicast extensive
Interface           State   Area            DR ID           BDR ID          Nbrs
xe-0/3/0.1503       PtToPt  0.0.0.0         0.0.0.0         0.0.0.0            1
  Address fe80::8ae0:f305:df55:f8f6, Prefix-length 64
  OSPF3-Intf-index 1, Type P2P, MTU 1500, Cost 1
  Adj count: 1, Router LSA ID: 0
  Hello 10, Dead 40, ReXmit 5, Not Stub
  <font color='red'>IPSec SA</font> name: <font color='red'>ospf3-sa</font>
  Protection type: None
</pre>                 
                
                
<p>
In my experience, I have never really had to troubleshoot this. 
The only thing I have encountered up until this point was a key-mismatch or a configuration type (none versus IPsec or whatever) mismatch. 
Both of these errors are quite easy to spot when they show up in the logging of the router. 
</p>
                 

