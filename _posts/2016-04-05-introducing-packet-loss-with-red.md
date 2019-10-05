---
layout: post
title: Introducing packet loss with RED
image: /img/juniper_logo.jpg
---

<p>
Recently, a customer had several issues going on at the same time. The customer had an MPLS L3VPN with a default route towards a central firewall in the datacenter. Behind this firewall, there was some rackspace and a cloud environment. A lot of components were involved and after solving the individual problems, I was interested in knowing just how much of the issue was caused by packet loss. And not just that, what I wanted was to be able to drop a certain amount of traffic, see what effect it had and then gradually increase the packet loss. I found that RED, Random Early Discard, was a very useful tool in this case. It allowed me to introduce random packet loss and control the exact amount of traffic that was to be dropped.
</p>

<p>
The first step was the creation of a drop-profile, like this:
</p>
<pre style="font-size:12px">set class-of-service drop-profiles drop_5_percent fill-level 0 drop-probability 5</pre>
<p>
Setting the fill-level to '0' makes sure that packets are dropped as soon as the queue is zero percent full.  
The effect of this is that 5% of the packets are discarded, no matter how low or high the link utilization is.
The next step was the creation of a scheduler-map that applies the drop-profile previously created:    
</p>
<pre style="font-size:12px">set class-of-service schedulers be-test transmit-rate percent 100
set class-of-service schedulers be-test drop-profile-map loss-priority any protocol any drop-profile drop_5_percent</pre>
<p>
Since we only want to drop 5% of the traffic in this test, we do not need any other schedulers or classes for this scenario. Therefore, we can create a scheduler-map with only 1 scheduler:    
</p>
<pre style="font-size:12px">set class-of-service scheduler-maps test forwarding-class best-effort scheduler be-test</pre>
<p>
The 'best-effort' is a reference to the default forwarding class that should exist on any Junos OS device right out of the box.
Applying the scheduler-map can be done in the following way:    
</p>
<pre style="font-size:12px">set class-of-service interfaces ge-1/1/9 unit 1520 scheduler-map test</pre>    
<p>
To be able to commit this configuration, we have to make sure that the interface is configured for a per-unit scheduler, like this:    
</p>
<pre style="font-size:12px">set interfaces ge-1/1/9 per-unit-scheduler</pre>    
<p>
    To quickly verify that random packets are being dropped, we can send some packets through the interface we just applied the scheduler-map to:
</p>

<pre style="font-size:12px">&lt;some-customer-cpe>ping -vpn-instance 1 -m 2 -c 1000 -s 1464 -f 10.0.10.1
  PING 10.0.10.57: 1464  data bytes, press CTRL_C to break
    Reply from 10.0.10.57: bytes=1464 Sequence=1 ttl=125 time=8 ms
..
.
    Reply from 10.0.10.57: bytes=1464 Sequence=996 ttl=125 time=8 ms
    Reply from 10.0.10.57: bytes=1464 Sequence=997 ttl=125 time=8 ms
    Request time out
    Reply from 10.0.10.57: bytes=1464 Sequence=999 ttl=125 time=9 ms
    Reply from 10.0.10.57: bytes=1464 Sequence=1000 ttl=125 time=8 ms

  --- 10.0.10.57 ping statistics ---
    1000 packet(s) transmitted
    941 packet(s) received
    5.90% packet loss
    round-trip min/avg/max = 8/8/10 ms</pre>
<p>
A nice and random 5% packet loss is the result. After checking the effect, I gradually increased the packet loss, simply by increasing the drop percentage in the drop-profile. I first changed it to 10 percent, then 25 and ultimately to 50 percent.    
</p>
<p>
In the previous example, the interface was configured with a per-unit scheduler. If the interface would have been configured with the ‘hierarchical-scheduler’ statement, the use of a traffic-control profile would have been necessary:    
</p>
<pre style="font-size:12px">set class-of-service traffic-control-profiles test scheduler-map test
set class-of-service traffic-control-profiles test shaping-rate 1g
set class-of-service interfaces ge-1/1/9 unit 1520 output-traffic-control-profile test</pre>
<p>
    Apart from sending pings or looking at the performance of an application running across the link, another way to verify that packets are being dropped would be by issuing the following command:
</p>
<pre style="font-size:12px">play@somemx80> show interfaces queue ge-1/1/9.1520 forwarding-class best-effort
  Logical interface ge-1/1/9.1520 (Index 598) (SNMP ifIndex 816)
Forwarding classes: 16 supported, 4 in use
Egress queues: 8 supported, 4 in use
Burst size: 0
Queue: 0, Forwarding classes: best-effort
  Queued:
    Packets              :                 10016                     0 pps
    Bytes                :              15421624                     0 bps
  Transmitted:
    Packets              :                  9569                     0 pps
    Bytes                :              14732350                     0 bps
    Tail-dropped packets :                     0                     0 pps
    RL-dropped packets   :                     0                     0 pps
    RL-dropped bytes     :                     0                     0 bps
    <font color='red'>RED-dropped packets</font>  :                   447                     0 pps
     Low                 :                   447                     0 pps
     Medium-low          :                     0                     0 pps
     Medium-high         :                     0                     0 pps
     High                :                     0                     0 pps
    <font color='red'>RED-dropped bytes</font>    :                689274                     0 bps
     Low                 :                689274                     0 bps
     Medium-low          :                     0                     0 bps
     Medium-high         :                     0                     0 bps
     High                :                     0                     0 bps</pre>

<p>
Another verification method would be issuing the command <b>show class-of-service interface ge-1/1/9.1520 comprehensive</b>. This will give you all the class of service related settings and counters for that specific interface.     
</p>

<p>
It’s a crude but quick and somewhat effective way to introduce some packet loss. 
It helped me and my customer understand how much loss some applications could tolerate and how things would be experienced by the end-users.    
</p>