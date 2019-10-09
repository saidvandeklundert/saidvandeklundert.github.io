---
layout: post
title: Juniper Virtual Chassis Fabric AE interface
tags: [juniper, qfx, vcf ]
image: /img/juniper_logo.jpg
---

<p>
Connecting other parts of the network to the <b>VCF</b> in a redundant way using <b>Link Aggregation Groups</b> (LAG) is very easy. 
A LAG can combine several Ethernet interfaces into a single logical link called an Aggregate Ethernet (<b>AE</b>) interface. 
When you are running a VCF, you’d best spread a LAG across multiple member switches. 
This way, during an NSSU or during the loss of 1 node, everything in your network will keep on running.
</p>

![Virtual Chassis Fabric LAG](/img/vcf_lag.png "Virtual Chassis Fabric LAG") 

              
<p>
Before we can get into the configuration of the AE interface itself, we need to start by enabling the VCF to support AE interfaces. 
This is done in the [<b>chassis aggregated-devices</b>] stanza:
</p>
<pre style="font-size:12px">
set chassis aggregated-devices ethernet device-count 20</pre>
<p>
    This command enables the VCF to support 20 AE interfaces. Without this configuration statement, configured AE interfaces will not be usable.
</p>
<p>
    After enabling the AE interface, we can now move on to the configuration of the interface itself. Let’s create a 20Gbit AE interface that is spread across members 3 and 6:
</p>
 

![Virtual Chassis Fabric AE example](/img/vcf_ae_interface_example.png "Virtual Chassis Fabric AE example") 

<p>
    After preparing the VCF for AE interfaces, let’s assign the physical interfaces to the proper LAG:
</p>
<pre style="font-size:12px">
delete interfaces xe-3/0/0
set interfaces xe-3/0/0 description to_some_other_switch
set interfaces xe-3/0/0 ether-options 802.3ad ae0

delete interfaces xe-6/0/0
set interfaces xe-6/0/0 description to_some_other_switch
set interfaces xe-6/0/0 ether-options 802.3ad ae0</pre>
<p>
    Because the interfaces that are part of an AE cannot have other options configured, using the ‘delete interfaces’ command quickly removes all other configuration. 
    The only things that needs to be configured for the physical interface is the ‘ether-options 802.3ad’ statement. 
    This contains a reference to the AE interface that the physical interface will belong to.
    In our case, this is AE0, since this will be the logical interface we want to start using.
</p>
<p>
    After assigning the physical interfaces to the AE, we can move on to the AE interface itself. Let’s create a trunk interface and make the individual physical interfaces send out LACP messages:
</p>
<pre style="font-size:12px">
set interfaces ae0 description to_some_other_switch
set interfaces ae0 aggregated-ether-options link-speed 10g
set interfaces ae0 aggregated-ether-options lacp active
set interfaces ae0 unit 0 family ethernet-switching interface-mode trunk
set interfaces ae0 unit 0 family ethernet-switching vlan members 100</pre>
<p>
    The above configuration will also work without the ‘aggregated-ether-options lacp’ statement. In that case, the AE would have been a static LAG and there would be no exchange of LACP control packets. 
</p>
<p>
    To verify that our LAG is working, we can issue the following command:
</p>
<pre style="font-size:12px">
play@VCF> show interfaces ae0 extensive
Physical interface: ae0, Enabled, Physical link is Up
  Interface index: 840, SNMP ifIndex: 682, Generation: 663
  Description: to_some_other_switch
  Link-level type: Ethernet, MTU: 1514, Speed: <font color='red'>20Gbps</font>, BPDU Error: None, MAC-REWRITE Error: None, Loopback: Disabled, Source filtering: Disabled, Flow control: Disabled, Minimum links needed: 1, Minimum bandwidth needed: 0
  Device flags   : Present Running
  Interface flags: SNMP-Traps Internal: 0x4000
  Current address: f0:1c:2d:48:a8:0d, Hardware address: f0:1c:2d:48:a8:0d
  Last flapped   : 2015-03-04 16:38:02 CET (16w4d 21:56 ago)
  Statistics last cleared: Never
  Traffic statistics:
   Input  bytes  :           1816375890                 3624 bps
   Output bytes  :           1499782977                 3432 bps
   Input  packets:             17792202                    3 pps
   Output packets:             13917070                    3 pps
   IPv6 transit statistics:
    Input  bytes  :                   0
    Output bytes  :                   0
    Input  packets:                   0
    Output packets:                   0
  Input errors:
    Errors: 0, Drops: 0, Framing errors: 0, Runts: 0, Giants: 0, Policed discards: 0, Resource errors: 0
  Output errors:
    Carrier transitions: 2, Errors: 0, Drops: 0, MTU errors: 0, Resource errors: 0
  Egress queues: 12 supported, 5 in use
  Queue counters:       Queued packets  Transmitted packets      Dropped packets
    0 best-effort                    0              7117509                    0
    3 fcoe                           0                    0                    0
    4 no-loss                        0                    0                    0
    7 network-cont                   0              6794564                    0
    8 mcast                          0                 5008                    0
  Queue number:         Mapped forwarding classes
    0                   best-effort
    3                   fcoe
    4                   no-loss
    7                   network-control
    8                   mcast

  Logical interface ae0.0 (Index 589) (SNMP ifIndex 695) (Generation 252)
    Flags: SNMP-Traps 0x20024000 Encapsulation: Ethernet-Bridge
    Statistics        Packets        pps         Bytes          bps
    Bundle:
        Input : 3353214937388021592          0 5059842287152            0
        Output: 5046586572800          0 1184550924416841968            0
    Link:
      xe-3/0/0.0
        Input :             0          0             0            0
        Output:             0          0             0            0
      xe-6/0/0.0
        Input :             0          0             0            0
        Output:             0          0             0            0
    <font color='red'>LACP info</font>:        Role     System             System      Port    Port  Port
                             priority          identifier  priority  number   key
      xe-3/0/0.0     Actor        127  f0:1c:2d:48:a7:00       127      12    14
      xe-3/0/0.0   Partner        127  2c:21:72:a3:70:40       127      21    11
      xe-6/0/0.0     Actor        127  f0:1c:2d:48:a7:00       127      13    14
      xe-6/0/0.0   Partner        127  2c:21:72:a3:70:40       127      22    11
    <font color='red'>LACP Statistics:       LACP Rx     LACP Tx</font>   Unknown Rx   Illegal Rx
      xe-3/0/0.0           3402935     3394454            0            0
      xe-6/0/0.0           3402969     3394276            0            0
    Marker Statistics:   Marker Rx     Resp Tx   Unknown Rx   Illegal Rx
      xe-3/0/0.0                 0           0            0            0
      xe-6/0/0.0                 0           0            0            0
    Protocol eth-switch, MTU: 1514, Generation: 292, Route table: 6
      Flags: Trunk-Mode
</pre>    
<p>
    The more interesting things to look for are the LACP info and the LACP statistics.
    LACP info displays the ‘Actor’ (local device) and the ‘Partner’ (remote device). 
    The LACP statistics will show you how many LACP hellos have been send and received.
</p>
<p>
    Anyway, this redundant interface is acting as a single 20Gbit link. It will now remain active as a 10Gbit link when a single member fails or whenever you initiate an NSSU (during an NSSU, individual members are rebooted one by one).
</p>
<p>
    Another important thing to realize is that by default, the VCF uses the ‘<b>layer2-payload</b>’ hash mode. This means that IPv4 and IPv6 payload fields are used to hash traffic on the LAG. You can verify the current hash setting by issuing the ‘show forwarding-options enhanced-hash-key’ command. 
    By fiddling about in the [ <b>forwarding-options enhanced-hash-key</b> ] configuration stanza, you can alter the default hashing behavior.
    The VCF has a lot of options that you can alter.
</p>
<p>
    The links between the individual VCF members (VCP’s) can also be bundled into a LAG. This will happen automatically when same-speed VCP links are configured between two devices.
    Load-sharing will also be active across these VCP LAGs.
</p>