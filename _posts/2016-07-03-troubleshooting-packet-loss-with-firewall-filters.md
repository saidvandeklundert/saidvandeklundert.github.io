---
layout: post
title: Troubleshooting packet loss with firewall filters
image: /img/juniper_logo.jpg
---

Troubleshooting packet loss with firewall filters
=================================================


Packet loss can be caused by all sorts of reasons. Could be faulty hardware, a software issue on a device, a congested link or some policers and shapers that are working against you. In order to fix packet loss in a network, you first have pinpoint where the packets are being dropped. Pinpointing where the packets are dropped is not always easy, especially if the packet loss is low, intermittent and affecting only 1 traffic flow specifically.

Suppose that a VM, sitting behind several switches and a router, is experiencing packet loss:


![tshoot with firewall filter](/img/tshoot_fw_filter.png "tshoot with firewall filter")

Using ping could tell us about the packet loss, but it could not tell us where the packets are dropped. We could ping the device that acts as a gateway to the VM, to see if packets make it up until that point. Problem is that the router in question might not feel like replying. This could be because of certain control plane protection measures that police ICMP or this could simply be because the router is busy doing something else.

Another option is that the router replies 100% of the time. If this is the case, you still don’t know whether or not packets can make it through to the egress interface. This is because traffic directed towards the router is exception traffic. This is not handled in the same way transit traffic is.

So how can you verify that the packets you send to the VM actually made it to the egress interface of the router? And after that, how can you check to see of the packets were forwarded by each individual switch?

In such cases, an additional tool to help isolate the problem could be the use of firewall filters. You can simply have a firewall filter count the ICMP packets send towards a certain destination IP address. You can even do this on a switch configured for layer 2 only.

Let’s see how we could do this on an MX router. The following configuration creates a firewall filter that matches ICMP and the IP address of the VM:

<pre>
set firewall family inet filter icmp-test term icmp from destination-address 1.1.1.1/32
set firewall family inet filter icmp-test term icmp from protocol icmp
set firewall family inet filter icmp-test term icmp then count icmp-testing
set firewall family inet filter icmp-test term icmp then accept
set firewall family inet filter icmp-test term accept-rest then accept
</pre>

The ICMP term accepts ICMP packets destined to 1.1.1.1. Whenever a packet is accepted, a counter called 'icmp-testing' is incremented. The last term matches and accepts all other traffic. Now, let’s apply the firewall filter:

<pre>
set interfaces ae1 unit 200 family inet filter output icmp-test
</pre>

If we then send 100 packets towards the VM, we can verify that the packets made it to the egress interface by issuing the following command:

<pre>
play@mx> show firewall filter icmp-test counter icmp-testing

Filter: icmp-test
Counters:
Name                                                Bytes              Packets
icmp-testing                                         8400                  100
</pre>

This way, we can see that packets made it through the chassis of the MX and reached the egress interface. At least up until the point where the filter is evaluated. If the count would not have been 100, it would be a good idea to move the firewall filter to the upstream interface away from the VM and apply the same filter in the inbound direction.

Between the router and the VM, we can also see several switches configured for layer 2 forwarding only To examine if packets made it to the switch right after the router, we can create a firewall filter like this:

<pre>
set firewall family ethernet-switching filter icmp-test term icmp from ip-destination-address 1.1.1.1/32
set firewall family ethernet-switching filter icmp-test term icmp from ip-protocol icmp
set firewall family ethernet-switching filter icmp-test term icmp from user-vlan-id 200
set firewall family ethernet-switching filter icmp-test term icmp then accept
set firewall family ethernet-switching filter icmp-test term icmp then count icmp-count
set firewall family ethernet-switching filter icmp-test term accept-rest then accept
</pre>

This firewall filter basically does the same thing and it can be applied in the input direction of the trunk interface like so:

<pre>
set interfaces ae0 unit 0 family ethernet-switching filter input icmp-test
</pre>

After sending some ICMP packets to the VM, we can examine the counter on the switch by using the following command:

<pre>
play@vcf> show firewall

Filter: icmp-test
Counters:
Name                                                Bytes              Packets
icmp-count                                         152200                  100
</pre>

This verifies that all the packets we send were received on input. By moving the firewall filter from one switch to the other, you can verify whether or not packets actually made it to the switch.

The previous filter works on QFX (or VCF). But if one of the switches in the layer 2 path would have been an EX4200, the configuration would have been a little different. The previous firewall filter would have looked like this on the EX4200:

<pre>
set firewall family ethernet-switching filter icmp-test term icmp from dot1q-tag 200
set firewall family ethernet-switching filter icmp-test term icmp from destination-address 1.1.1.1/32 
set firewall family ethernet-switching filter icmp-test term icmp from protocol icmp
set firewall family ethernet-switching filter icmp-test term icmp then accept
set firewall family ethernet-switching filter icmp-test term icmp then count icmp-count
set firewall family ethernet-switching filter icmp-test term accept-rest then accept
</pre>

Applying the firewall filter and checking the counter would have been no different. EX3200 and up should support this. Unfortunately, the EX2200 does not support count in the same way the other switches do (an overview of EX match conditions is found [here](https://www.juniper.net/documentation/en_US/junos/topics/reference/general/firewall-filter-ex-series-match-conditions-support.html).

Hope this helps!

