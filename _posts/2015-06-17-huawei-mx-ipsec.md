---
layout: post
title: Site-to-Site IPsec VPN between Huawei AR and Juniper MX
tags: [juniper ]
image: /img/juniper_logo.jpg
---


<p>
Today I configured an IPsec VPN between a Huawei AR1220F and a Juniper M104. I wanted to keep the configuration around for future reference.
The configuration on a Huawei is rather straightforward. To put the Huawei AR IPsec configuration in a picture:
</p>
             

![Huawei AR IPsec](/img/huawei_ipsec_config.png "Huawei AR IPsec")   

<p>
I’ve tried and tested this configuration using the following setup:
</p>


![Huawei AR IPsec scenario](/img/huawei_ipsec_scenario.png "Huawei AR IPsec scenario")   

   
<p>
Now the configuration to copy-paste from:
</p>
                
                
                
<pre style="font-size:12px">
# 
acl number 3000
 description ipsec-allowed-source-destination
 rule 10 permit ip source 192.168.168.0 0.0.0.255 destination 192.168.167.0 0.0.0.255
#
ipsec proposal p1
 esp authentication-algorithm sha1
 esp encryption-algorithm aes-256
#
ike proposal 1
 encryption-algorithm aes-cbc-256
 dh group2
 sa duration 12000
#
ike peer mx104 v1
 pre-shared-key simple ar1220fmx104
 ike-proposal 1
 remote-address 172.31.255.10
#
ipsec policy ipsec-vpn 10 isakmp
 security acl 3000
 ike-peer mx104
 proposal p1
#
interface GigabitEthernet0/0/1.575
 description ipsec-test-LAN-side
 dot1q termination vid 575
 ip address 192.168.168.1 255.255.255.0
#
interface GigabitEthernet0/0/0.502
 description ipsec-test-WAN-side
 dot1q termination vid 502
 ip address 80.0.0.2 255.255.255.252
 ipsec policy ipsec-vpn
#
</pre>
<p>
Some commands to verify what the AR router is doing (or trying to do):
</p>
<pre style="font-size:12px">
display ike sa 
display ipsec sa brief
display acl 3000
display ike statistics v1 
display ipsec statistics esp
display ipsec global config</pre>
<p>
To debug:
</p>
<pre style="font-size:12px">
terminal monitor
terminal debugging
debugging ipsec all
debugging ike all
undo debugging all</pre>
<p>
The configuration used on the Juniper side of this scenario:
</p>
<pre style="font-size:12px">
set interfaces ms-0/0/0 unit 10 description ipsec-tunnel
set interfaces ms-0/0/0 unit 10 family inet unnumbered-address xe-1/2/0.550
set interfaces ms-0/0/0 unit 10 service-domain outside
set interfaces ms-0/0/0 unit 11 description ipsec-tunnel
set interfaces ms-0/0/0 unit 11 family inet
set interfaces ms-0/0/0 unit 11 service-domain inside

set interfaces xe-1/2/0 unit 550 description IPsec-source-test
set interfaces xe-1/2/0 unit 550 vlan-id 550
set interfaces xe-1/2/0 unit 550 family inet address 172.31.255.10/29

set interfaces lo0 unit 1684 description to-enable-ping-test
set interfaces lo0 unit 1684 family inet address 192.168.167.1/24

set services service-set ipsec-firewall next-hop-service inside-service-interface ms-0/0/0.11
set services service-set ipsec-firewall next-hop-service outside-service-interface ms-0/0/0.10
set services service-set ipsec-firewall ipsec-vpn-options local-gateway 172.31.255.10
set services service-set ipsec-firewall ipsec-vpn-rules to-ar1220f

set services ipsec-vpn rule to-ar1220f term ipsec from source-address 192.168.167.0/24
set services ipsec-vpn rule to-ar1220f term ipsec from destination-address 192.168.168.0/24
set services ipsec-vpn rule to-ar1220f term ipsec then remote-gateway 80.0.0.2
set services ipsec-vpn rule to-ar1220f term ipsec then dynamic ike-policy ike-ar1220f
set services ipsec-vpn rule to-ar1220f term ipsec then dynamic ipsec-policy ar1220f
set services ipsec-vpn rule to-ar1220f match-direction input

set services ipsec-vpn ike policy ike-ar1220f mode main
set services ipsec-vpn ike policy ike-ar1220f proposals ike-ar1220f
set services ipsec-vpn ike policy ike-ar1220f pre-shared-key ascii-text "$9$Uoji.fT36CtPf1REhrlLxNds4ikP5z3UDAp"

set services ipsec-vpn ipsec proposal ar1220f protocol esp
set services ipsec-vpn ipsec proposal ar1220f authentication-algorithm hmac-sha1-96
set services ipsec-vpn ipsec proposal ar1220f encryption-algorithm aes-256-cbc

set services ipsec-vpn ike proposal ike-ar1220f authentication-method pre-shared-keys
set services ipsec-vpn ike proposal ike-ar1220f dh-group group2
set services ipsec-vpn ike proposal ike-ar1220f authentication-algorithm sha1
set services ipsec-vpn ike proposal ike-ar1220f encryption-algorithm aes-256-cbc
set services ipsec-vpn ike proposal ike-ar1220f lifetime-seconds 86400

set routing-instances ipvpn instance-type vrf
set routing-instances ipvpn interface ms-0/0/0.11
set routing-instances ipvpn interface lo0.1684
set routing-instances ipvpn route-distinguisher 1:1
set routing-instances ipvpn vrf-target target:1:1
set routing-instances ipvpn vrf-table-label
set routing-instances ipvpn routing-options static route 192.168.168.0/24 next-hop ms-0/0/0.11
</pre>   

          
