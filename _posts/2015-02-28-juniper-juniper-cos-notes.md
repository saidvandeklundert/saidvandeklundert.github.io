---
layout: post
title: Juniper cos notes
image: /img/juniper_logo.jpg
---

               
<p>
    These notes cover CoS on Juniper devices. The list of topics covered here correspond to the JNCIP-SP exam objectives. One objective is missing. I will cover the 'Given a scenario, demonstrate knowledge of how to configure and monitor CoS' somewhere else.
</p>

<p>
    The list of topics:
    <br>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#1">CoS processing on Junos devices</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#2">CoS header fields</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#3">Forwarding classes</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#4">Classification</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#5">Packet loss priority</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#6">Policers</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#7">Schedulers & drop profiles</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#8">Shaping</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#9">Rewrite rules</a>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <a class="tocxref" href="#10">Hierarchical scheduling</a>
</p>

<br><br>
<a name="1"></a>
<h3>
    CoS processing on Junos devices.
</h3>                
<p>
    On a Juniper device, the following CoS stages can be identified:
</p>

![Juniper cos IPv6 header](/img/junos_cos_processing.png "Juniper cos IPv4 header")  

<b>BA classifier</b>: on ingress, all traffic is classified based on the BA code point value (IPprec, DSCP, EXP or 802.1p).<br>
<b>Ingress policing</b>: discarding packets within a traffic stream on ingress to enforce a limit.<br>
<b>Multifield classifier</b>: setting the forwarding class and the loss-priority for packets by using a firewall filter to classify traffic based on one or more fields. <br>
<b>Forwarding policy</b>: CoS based forwarding.<br>
<b>Fabric</b>: how traffic is handled inside the chassis.<br>
<b>Egress policing</b>: discarding packets within a traffic stream on egress to enforce a limit.<br>
<b>Multifield classifier</b>: again, setting the forwarding class and the loss-priority for packets by using a firewall filter to classify traffic based on one or more fields.<br>
<b>Shaper</b>: delaying packets within a traffic stream going out an interface to a desired rate.<br>
<b>Scheduler(s)</b>: defines how different queues treat traffic. Scheduling encompasses the following components:
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; transmission rate: amount of bandwidth allocated to a queue
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; priority: access to the bandwidth compared to other queues or schedulers
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; delay buffer: data that can be stored when congestion occurs
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; RED drop profile: Random Early Detect, a congestion avoidance mechanism. RED monitors queue sizes and it can drop packets when queues fill up to a certain size.
<b>Rewrite marker</b>: rewrite of a CoS field for a next node.<br>
                
                
                
<br><br>
<a name="2"></a>
<h3>
    CoS header fields.
</h3>
<p>
    Typically, traffic is marked at the edge of your network. The rest of the network can handle traffic based on these markings. The way that traffic can be marked is by setting certain bits in various packet headers. The fields most commonly used for this are the following:
    IPv4 header:
</p>           

![Juniper cos IPv6 header](/img/qos_ipv4_header.png "Juniper cos IPv4 header")    

![Juniper cos IPv6 header](/img/qos_ipv6_header.png "Juniper cos IPv6 header")                

![Juniper cos L2 header](/img/qos_l2_header.png "Juniper cos L2 header")            


![Juniper cos MPLS header](/img/qos_mpls_header.png "Juniper cos MPLS header")


<br><br>  
<a name="3"></a>
<h3>
    Forwarding classes.
</h3>                
<p>
    Forwarding classes can be thought of as the configuration component that represents the queues. 
    The classifiers, either multifield or BA, assign the traffic to a forwarding class. 
    Each forwarding class is mapped to a queue.
</p>             

![Juniper cos forwarding class](/img/junos_cos_forwarding_class.png "Juniper cos forwarding class")

<p>
    By default, Junos has a total of four forwarding classes defined. In most cases, without any manipulation, all non-RE generated packets will end up in the best-effort forwarding class. RE-generated packets (OSPF, BGP, etc) will end up in the network-control forwarding class.
    Different routing platforms offer a different number of queues. 
    Apart from the four default queues, possible other queues are unused when they are not manually configured.                    
</p>
<p>
    Below is a part of the output of the ‘show class-of-service interface xe-2/0/0 comprehensive’ command. It shows the four default queues and the percentage of bandwidth that is allocated to the queues.
</p>
<pre class="prev"> 
play@MX104> show class-of-service interface xe-2/0/0 comprehensive
...
output omitted
...
  Egress queues: 8 supported, 4 in use
  Queue counters:       Queued packets  Transmitted packets      Dropped packets
    0 best-effort           1380441441           1380441441                    0
    1 expedited-fo                   0                    0                    0
    2 assured-forw                   0                    0                    0
    3 network-cont            66224351             66224351                    0
  Queue number:         Mapped forwarding classes
    0                   best-effort
    1                   expedited-forwarding
    2                   assured-forwarding
    3                   network-control
 ..
 output omitted
 ..
  CoS information:
    Direction : Output
    CoS transmit queue               Bandwidth               Buffer Priority   Limit
                              %            bps     %           usec
    0 best-effort            95     9500000000    95              0      low    none
    3 network-control         5      500000000     5              0      low    none
 ..
 output omitted
 ..
</pre>
<br><br>
<a name="4"></a>
<h3>
    Classification.
</h3>    
<p>
    Junos offers different methods for classification.
    There is the fixed classification, the BA classification and the multifield classifier option.
</p>
<p>
    <b>
        Fixed classification:
    </b>
    fixed classification can be used to classify all traffic received on a logical interface. It is simple, efficient and offers no flexibility whatsoever.
</p>
<p>
    <b>
        BA classification:
    </b>
    BA classification can be used when the markings of neighboring nodes can be trusted. BA classification examines the existing markings in packets received. Based on these markings, the BA classifier will make sure that the received packets will be placed in the appropriate forwarding class. BA classification is applied before the MF classification. The MF classification can override the BA classification.
</p>
<p>
    BA classification can be configured for logical interfaces and a single logical interface can be assigned multiple BA classifiers.
</p>
<p>
    <b>
        MF classification:
    </b>
    The most granular method of classifying traffic on Junos requires the creation of a firewall filter. This firewall filter can reference interesting traffic and use the ´then´  action to place the traffic in the relevant forwarding class and mark the packets accordingly. 
    The from action of the firewall filter makes it possible to isolate interesting traffic based on a whole variety of characteristics. 
    This includes a source or a destination IP address, a source of a destination port or a protocol.
</p>                
<p>
    The MF classifier can  be used on input and on output. When the firewall filter is applied in the outbound direction of the loopback interface, RE-generated traffic can be classified as well.
</p>
<br><br>
<a name="5"></a>
<h3>
    Packet loss priority.
</h3>                 
<p>
    The packet loss priority (PLP) identifies the likelihood a packet will be dropped when there is congestion. 
    The PLP is assigned to all ingress packets. The PLP can be assigned by using a BA classifier, a MF classifier or a policer.
</p>
<p>
    There are four possible PLP’s:
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; low
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; medium-low
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; medium-high
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; high
</p>
<p>
    The default Junos classifiers will only assign packets to either low or high.
</p>
<p>
    Example of the default PLP on a MX104:
</p>
<pre class="prev">  
Classifier: ipprec-compatibility, Code point type: inet-precedence, Index: 13
  Code point         Forwarding class                    Loss priority
  000                best-effort                         low
  001                best-effort                         high
  010                best-effort                         low
  011                best-effort                         high
  100                best-effort                         low
  101                best-effort                         high
  110                network-control                     low
  111                network-control                     high
</pre>

<br>
<br>
<a name="6"></a>
<h3>
    Policers, including tricolor marking and hierarchical policers.
</h3>                
<p>
    Policing is the process of discarding packets within a traffic stream to enforce a limit. Policing is done using the token bucket algorithm as opposed to the shaper’s leaky bucket.
</p>
           

![Juniper cos token bucket](/img/junos_qos_token_bucket.png "Juniper cos token bucket")


<p>
    The bucket is filled with tokens. 
    The rate at which this is done corresponds to the configured bandwidth rate. 
    When the bucket is full, tokens that are arriving are discarded. 
    For every packet that is transmitted, a token is removed from this bucket. 
    As soon as the bucket is depleted, traffic can be discarded or remarked to a lower priority. 
    There are two methods of policing, hard policing and soft policing. Hard policing is about discarding traffic. 
    Soft policing is about marking traffic down.
</p>
<p>
    Junos offers different types of policers:                    
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>Single-rate two-color</b>: soft or hard policing
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>Single-rate tricolor marking</b>: soft policing based on burst thresholds
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>Two-rate tricolor marking</b>: soft policing based on bandwidth thresholds
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>Hierarchical</b>: allows for separate policing of premium and aggregate traffic on the same interface
</p>
<p>
    Policer terminology:
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>CIR</b>: guaranteed traffic in bits per second
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>PIR</b>: maximum rate that can be sent through the policer in bits per second
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>CBS</b>: guaranteed burst-size in bytes
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>PBS</b>: maximum burst-size in bytes
</p>
<p>
    Policers can be color-blind or color aware. A color-blind policer will not take into consideration previous markings. A color-aware policer (default) will.
</p>
<p>
    In Junos, policers can be applied directly on the interface or through a firewall filter.
    When a policer is applied directly to an interface, it can be applied in the input or in the output direction. 
    On an MX, the policer can be applied per protocol family, per logical interface or per physical interface. 
    The policer configured under the interface can be a standard or a logical-interface-policer.
</p>
<p>
    When a ‘standard’ policer is configured under the protocol family, it will be a separate instance of that policer and it will police only traffic from that address family.  In the following example, both IPv4 and IPv6 have a separate policer installed in the forwarding table:
</p>
<pre class="prev">  
set firewall policer 100m-standard if-exceeding bandwidth-limit 100m
set firewall policer 100m-standard if-exceeding burst-size-limit 5m
set firewall policer 100m-standard then discard
set interface ge-0/0/8 unit 0 family inet policer input 100m-standard
set interface ge-0/0/8 unit 0 family inet6 policer input 100m-standard
</pre>    
<p>
    A logical interface policer will treat traffic for a logical interface as an aggregate, meaning that a single policer can police traffic for multiple families to a single desired rate. The logical interface policer is applied under the different address families but it acts as 1 instance of the policer per logical interface. The policer has to be configured with the ‘logical-interface-policer’ keyword.
    In the following example, both IPv4 and IPv6 have a combined policer configured:
</p>
<pre class="prev">  
set firewall policer 100m-standard logical-interface-policer
set firewall policer 100m-standard if-exceeding bandwidth-limit 100m
set firewall policer 100m-standard if-exceeding burst-size-limit 5m
set firewall policer 100m-standard then discard
set interface ge-0/0/8 unit 0 family inet policer input 100m-standard
set interface ge-0/0/8 unit 0 family inet6 policer input 100m-standard
</pre>                   
<p>
    Through a firewall filter, policers can be configured input or output. 
    When using a firewall filter to reference policers, it is important to understand the difference between ‘filter specific’ and ‘term specific’. 
    A ‘term specific’ policer is a separate instance of the policer for an individual term. 
    With a ‘filter specific’ policer, a single instance of a policer will police several terms. 
    The Junos default is ‘term specific’. To create a ‘filter-specific’ policer, the keyword ‘filter-specific’ has to be added to the policer.
</p>
<p>
    Using a firewall filter, you can create the following policers:
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>single-rate two-color policer</b>: configure the ‘if-exceeding bandwidth-limit’ in bits and a ‘if-exceeding burst-size-limit’ in bytes. A ‘then discard’ is used for hard policing, and a ‘then forwarding-class’ is used for soft policing
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>single-rate tricolor policer</b>: configure committed-information-rate <i>bps</i>, committed-burst-size <i>bytes</i> and excess-burst-size <i>bytes</i>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>two-rate tricolor policer</b>: configure committed-information-rate <i>bps</i>, committed-burst-size <i>bytes</i>, peak-information-rate <i>bps</i> and peak-burst-size <i>bytes</i>
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>physical interface policer</b>: allows the aggregate policing, under 1 interface, of different logical interfaces and different address families. A two or tricolor policer with the keyword ‘physical-interface-policer’ needs to be referenced in a firewall filter. The firewall filter can then be applied to the different logical interfaces and address-families.
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>hierarchical policer</b>: up next.
</p>
<p>
    The hierarchical policer, configured under the [firewall hierarchical-policer] stanza can differentiate between premium and non-premium traffic.
    It is an aggregate policer with rates defined for both premium and non-premium. 
    Premium traffic is allowed to grab a token from the bucket before non-premium traffic. 
    When the premium tokens are drained from the bucket, premium traffic is discarded. 
    Non-premium traffic can grab premium tokens as long as there is no premium traffic. 
</p>
<p>
    The premium traffic can be classified by the BA or the MF. The designation premium traffic is given to a forwarding class in the [class-of-service] stanza.
</p>
<p>
    Last policing note when you want to police ae interfaces on an MX. 
    It is important to understand the chassis you are working on. 
    When all ae-member interfaces belong to the same PFE, traffic will be policed at the referenced bandwidth.
    When ae-member links are spread across different PFE’s, there will be a policer per PFE. 
    For instance, when you apply a 100m policer to a 2 member ae that is spread across 2 PFE’s, the traffic on each individual ae member will be policed at a rate of 100m. This can be mitigated by using the keyword ‘shared-bandwidth-policer’ keyword inside the policer configuration. This will make the policer behave as an aggregate policer, regardless of how many PFE’s are involved (unfortunately, no ‘shared-bandwidth-policer’ on EX or QFX).
</p>

<br><br>
<a name="7"></a>
<h3>
    Schedulers and drop profiles.
</h3>
<p>
    Listed as separate topics for JNCIP. I put them together since a drop-profile is referenced by a scheduler and it just made more sense to me. A scheduler-map can connect different forwarding-classes to schedulers and each scheduler can have different characteristics:
</p>
                

![Juniper cos scheduler](/img/junos_cos_scheduler.png "Juniper cos scheduler") 

  
<p>
    The components of an individual scheduler are the following:
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; transmission rate: this determines the minimum amount of guaranteed bandwidth that is allocated to a queue.
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; priority: this determines the scheduling priority that is applied to the queue
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; delay buffers: the size of the queue buffer
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; RED drop profiles: controls how traffic is dropped when congestion occurs 
    
</p>
<p>
    <b>
        Transmission rate:
    </b>
</p>
<p>
    The transmission rate sets the amount of bandwidth that is allocated to a queue. The rate is specified in bits per second. Juniper allows for the following options to be added to the transmit-rate:
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>rate</b>: transmission rate in bps
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>percentage</b>: a percentage of the total available bandwidth (can be determined by the interface bandwidth or the shaper of the traffic-control-profile) 
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>remainder</b>: allows the queue to use all remaining available bandwidth
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>exact</b>: enforces the transmission rate in the way a shaper would. Statement can be combined with rate or percentage. When it is configured, an exact transmission rate or percentage is enforced. Without the keyword exact, queues can endlessly burst above the transmission rate as long as other queues are empty. 
    <br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>rate-limit</b>: enforces the transmission rate in the way a policer would. With or without congestion, the queue will never be able to send more than the transmission rate that is specified                    
</p>
<p>
    By default, MX and EX will share excess bandwidth in the ratio of the transmit rates. 
</p>
<p>
    <b>
        Priority:
    </b>
</p>  
<p>
    Five levels of transmission priority exist; low, medium-low, medium-high, high and strict-high. The queues are serviced in a descending order. The higher priority queues are serviced for as long as they are in profile. The exception is the strict-high queue. As long as the strict-high queue has traffic to send, it receives precedence over all other queues. The strict-high queue can be seen as a low-latency queue.
</p>
<p>
    <b>
        Delay buffers:
    </b>
</p>  
<p>
    There are two congestion control mechanisms at the output stage, the delay buffers and RED. The delay buffers determine how much traffic can be stored at times of congestion. When a delay buffer is full, packets will be dropped.
    A large buffer that is continuously filled will translate to a high delay. The buffer-size can be configured in three different ways.
</p>
<p>
    You can define is as a percentage of the port's total buffer. 
    In this case, the memory allocation dynamic (MAD) mechanism is used. 
    MAD can allocate additional buffer sizes to a queue when the transmit rate of at least one other queue is underutilized. 
    This MAD mechanism makes the buffers more elastic as they are able to expand to absorb a heavy burst. 
    When all queues are taking full advantage of the permitted transmit rate, MAD has no effect.
</p>
<p>
    The buffer can also be defined as a temporal value (in ms). When a temporal value is configured, MAD is not enabled and if traffic is exceeding the buffer size, it will be dropped.
</p>
<p>
    There is a third option, this is by using the keyword 'remainder'. This will simply allocate all leftover buffer size to the queue.
</p>
<p>
    Maximum buffer sizes on Juniper devices are very platform dependent. 
</p>
<p>
    <b>
        RED drop profiles:
    </b>
</p>    
<p>
    In Junos, Random Early Detect is configured using a drop-profile. The RED mechanism is about dropping packets before the delay buffer is 100% full. Instead for queues to fill up and start tail-dropping (allowing for TCP global synchronization), RED will start to drop packets before the queue is filled up. This will effectively hold back certain TCP connections in an attempt to increase the total throughput.
</p>
<p>
    RED can be configured to increase the drop probability as the buffer-size fills up. Through configuration options, RED can make distinctions between the packet loss priority of the different packets that are being queued. 
</p>
<p>
    An important consideration for Junos' drop-profile is the segmented versus the interpolated style. The segmented style, the default, will enforce the configured drop levels. The interpolated drop profile, using the keyword 'interpolate', will make Junos use the configured values to create an interpolated graph. 
</p>
<p>
    The following is a configuration example of a scheduler that is called ‘scheduler-example’:
</p>                 
<pre class="prev"> 
set class-of-service schedulers scheduler-example transmit-rate percent 70
set class-of-service schedulers scheduler-example buffer-size percent 70
set class-of-service schedulers scheduler-example priority medium-high
set class-of-service schedulers scheduler-example drop-profile-map loss-priority low protocol any drop-profile red-example-low
set class-of-service drop-profiles red-example-low fill-level 30 drop-probability 25
set class-of-service drop-profiles red-example-low fill-level 50 drop-probability 50
set class-of-service drop-profiles red-example-low fill-level 75 drop-probability 75
set class-of-service drop-profiles red-example-low fill-level 95 drop-probability 100
</pre>     
<p>
    A scheduler-map can be used connect schedulers to queues (or forwarding classes). Below is an example of a scheduler-map that ties three schedulers to three forwarding classes:
</p>
<pre class="prev"> 
set class-of-service scheduler-maps MAP forwarding-class be scheduler be-scheduler
set class-of-service scheduler-maps MAP forwarding-class bronze scheduler bronze-scheduler
set class-of-service scheduler-maps MAP forwarding-class gold scheduler gold-scheduler
</pre>                  
<p>
This scheduler-map can be applied directly to the interface or it can be applied via a traffic-control profile.
</p>
<pre class="prev"> 
set class-of-service interfaces ge-0/0/* scheduler-map MAP
</pre>                  
    
    

<br><br>
<a name="8"></a>
<h3>
Shaping.
</h3> 
<p>
Shaping is about delaying packets within a traffic stream going out an interface to a desired rate. Shaping uses the leaky bucket algorithm.
</p>          

![Juniper cos leaky bucket](/img/junos_qos_leaky_bucket.png "Juniper cos leaky bucket") 
        
<p>
In Junos, traffic shaping can be configured in multiple ways. A traffic shaper can be configured inside a scheduler, on an interface and it can be configured inside a traffic-control profile (configuration template used for hierarchical CoS).
</p>
<p>
Shaping inside a scheduler: when you configure the transmit rate of a scheduler, you can use the keyword ‘exact’. This keyword will make the scheduler install a shaper for that queue.
</p>                
<pre class="prev"> 
set class-of-service schedulers scheduler-example transmit-rate percent 30 exact
</pre>                          
<p>
Shaping on an interface: a pretty straightforward way of installing a shaper. 
</p>                
<pre class="prev"> 
set class-of-service interfaces ge-1/1/2 shaping-rate 10000000
</pre>     
<p>
Shaping inside a traffic control-profile:  you can have a traffic-control profile reference a shaper and a scheduler map. 
</p>
<pre class="prev"> 
set class-of-service traffic-control-profiles 100m scheduler-map example
set class-of-service traffic-control-profiles 100m shaping-rate 100m
</pre>                                          
<p>
The traffic-control profile is then applied to an interface, like this:
</p>                
<pre class="prev"> 
set class-of-service interfaces ge-1/1/2.500 output-traffic-control-profile 100m
</pre>           
<p>
Or, in case of when you want to apply cos to subscribers:
</p>
<pre class="prev"> 
set dynamic-profiles subscriber-profile class-of-service traffic-control-profiles pppoe scheduler-map "$junos-cos-scheduler-map"
set dynamic-profiles subscriber-profile class-of-service traffic-control-profiles pppoe shaping-rate "$junos-cos-shaping-rate"
set dynamic-profiles subscriber-profile class-of-service interfaces pp0 unit "$junos-interface-unit" output-traffic-control-profile pppoe
</pre>           
                
<br><br>
<a name="9"></a>
<h3>
Rewrite rules.
</h3>             
<p>
The final and last stage of CoS processing in Junos is the rewrite marker stage. 
In this stage, rewrite rules can be implemented to change the value of the CoS bits of packets leaving the device on egress. 
The rewrite rules allow for the manipulation of IPv4/IPv6 DSCP, IPprec, MPLS EXP and IEEE802.1p bits. 
What the rewrite rule does is check the current forwarding class and PLP of a packet, perform a lookup in the rewrite table and apply that CoS value to the packet leaving the router. 
</p>
<p>
You can apply multiple rewrite rules to a single logical interface and a rewrite rule can manipulate the CoS bits of multiple headers. For example, you could implement a rewrite rule that sets both DSCP and EXP bits for traffic from a certain forwarding class.
</p>
<p>
There are default rewrite-rules defined, but none of those are actually applied. 
</p>

<br>
<a name="10"></a>
<h3>
Hierarchical scheduling (H-CoS) characteristics (high-level only).
</h3>                 
<p>
There is port level CoS and there is H-CoS. H-CoS offers additional levels of scheduling hierarchy as well as more in-depth queuing options. It allows for tiered shaping, scheduling and queuing.
</p>
<p>
H-CoS procides you with four levels of CoS:
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 1: physical interface
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 2: interface set (vlan or logical interface group)
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 3: logical interface (vlan)
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 4: queue
</p>
<p>
Example for each level:
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 1: shaper on the interface using a traffic control profile
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 2: a logical interface with a dual-tagged vlan where each inner tag is offered individual shaping and/or scheduling 
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 3: a logical interface with a (single) vlan can be offered shaping or/and scheduling
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; level 4: control over the queues that schedule traffic flows (scheduler map: transmit rate, RED, etc.)
</p>
<p>
A traffic control profile is the configuration template that is used to specify scheduler and queuing properties. In the H-CoS context, it can be applied to a physical interface, an interface set and to logical interfaces. 
</p>
<p>
On a juniper MX, if you have the proper hardware, you can choose to enable one of the following hierarchical scheduler modes:
</p>
<p>
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>per unit scheduler:</b>
<br>
This H-CoS mode does not allow for interface-sets. It will enable you to shape the interface, vlans configured on the interface and it offers a scheduler to every vlan configured on the interface.
</p>                             

![Juniper cos per unit](/img/junos_cos_per-unit.png "Juniper cos per unit") 

<p>
When an interface is configured with the ‘per-unit-scheduler’, you can shape on the port level and control scheduling and queuing on the vlan level. Control over the interface-set, or level 2 H-CoS, is not offered. 
</p>
<p>
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; <b>hierarchical scheduler:</b>
<br>
The hierarchical scheduler operates on all four H-CoS levels and gives you control over the interface-sets;
</p>                                                    

![Juniper cos hierarchical](/img/junos_cos_hierarchical.png "Juniper cos hierarchical") 

<p>
When an interface is configured with the ‘hierarchical-scheduler’, you can do more. You can shape on the port level and on the vlan level, have access to interface-sets, control scheduling and queuing on the vlan level and share them among vlans. 
</p>
<p>
For logical units that were not configured with a traffic control profile, you can configure a traffic control profile using the ‘output-traffic-control-profile-remaining’ keyword. This can then be applied to an interface or an interface set. All traffic sent through a vlan or interface-set without a traffic-control profile will be subjected to this traffic-control-profile.
</p>
<p>
In H-Cos, the scheduling mechanism used by Junos is PQ-DWRR, or Priority Queue Deficit Weighted Round Robin. The queues are emptied in the following order;
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; strict-high and high priority queues of all Vlans and interface-sets
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; medium-high and medium low priority queues
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; low priority queues
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; excess priority queues
</p>
<p>
The bandwidth that remains after all guaranteed bandwidth is allocated is called excess bandwidth. The excess bandwidth can be shared in one of three ways;
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; proportional to CIR
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; proportional to PIR
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; equal sharing
</p>
<p>
The node can be configured to calculate the remaining bandwidth in two ways;
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; per port
<br>&nbsp;&nbsp;&nbsp;&nbsp;&bull; per interface-set
</p>
<p>
The delay buffers in H-CoS offer no dynamic memory allocation. The buffer size can be configured explicitly under the traffic control profile, or the system can calculate it based on the VLANs guaranteed rate or shaping rate. Buffer size allocation is supported on level 1, 2 and 3.
</p>
<p>
RED performs the same function in H-CoS. It is recommended to only use the segmented-style-profile. In this segmented-style-profile, you should configure two entries. A drop probability starting point of 0 and a second value of whatever you want. The system will enforce a linear increase in drop-probability between the two configured values.
</p>
                                  
                
                
                
                
