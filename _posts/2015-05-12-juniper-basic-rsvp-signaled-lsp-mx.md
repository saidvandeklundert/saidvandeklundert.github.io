---
layout: post
title: Basic RSVP signaled LSP on MX
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---
   

<p>
This article is about the basic configuration on how to get an RSVP signaled MPLS LSP (label-switched path) working on a Juniper MX router.
The focus will be on the minimum amount of configuration needed to create LSPs between the Tiberius and the Commodus router:
</p>            
          

![RSVP basic configuration](/img/rsvp-the-basic-configuration-1.png "RSVP basic configuration") 

<p>
After we have established the LSPs, we’ll proceed and route some traffic across the LSPs. 
After verifying IP connectivity across the LSP’s, we’ll proceed and have a more detailed look into the control plane operation of this setup.
At the end of the article, I'll post the complete configuration for this lab.
</p>
<p>
The topology is rather straightforward. There are 8 Juniper MX routers involved and all of them are running IS-IS. I’ll skip a detailed description of the IS-IS configuration and only post the configuration that is applied to Tiberius:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show configuration protocols isis</b>
level 2 {
    authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; ## SECRET-DATA
    authentication-type md5;
    wide-metrics-only;
}
interface xe-0/2/0.9 {
    point-to-point;
    level 1 disable;
}
interface lo0.9 {
    level 1 disable;
}                    
</pre>
<p>
All the routers have a similar IS-IS configuration applied to make sure that there is IP-connectivity between all the routers. 
	Apart from having a NET address, the routers also have the iso family activated on all the links running IS-IS.
</p>
<p>
Before starting the MPLS configuration, there is one thing I would like to point out. The configuration command for ‘traffic-engineering’ under IS-IS is not there. This is because on MX routers, traffic-engineer is enabled by default for IS-IS. IS-IS has support for TLVs that can be used to distribute path information so that RSVP can do its work. If OSPFv2 would have been used, the ‘traffic-engineering’ configuration statement would be needed. This is because OSPF does not support traffic-engineering by default.
</p>
<p>
After configuring IS-IS on all the routers, the next thing we need to do is to enable the mpls family on all the interfaces we want to be able to handle the MPLS traffic.
Additionally, those same interfaces will need to be configured under the [protocols mpls] stanza as well.
</p>
<p>
As an example, let’s look in to Tiberius. This router is running IS-IS across the xe-0/2/0.9 link with Hadrian. This same link is going to need to handle MPLS traffic. To enable this link for MPLS traffic, we only need to issue the following two configuration commands:
</p>           

![RSVP basic configuration](/img/rsvp-the-basic-configuration-2.png "RSVP basic configuration") 
  
<p>
The configuration of the RSVP part is not that extensive either. 
All the links that are enabled for IS-IS and MPLS need to be added to the [protocols rsvp] stanza. 
Taking the Tiberius router as the example router again, we enable RSVP on the link in the following way:
</p>


![RSVP basic configuration](/img/rsvp-the-basic-configuration-3.png "RSVP basic configuration") 

<p>
With the commands displayed in the picture above, we have enabled RSVP on the interface of the Tiberius router. 
By adding the ‘authentication-key’ to the configuration statement, we have also enabled HMAC-MD5 message-based digest based authentication. 
This will authenticate all RSVP protocol exchanges between the Tiberius and the Hadrian router.
We can verify that the link is RSVP enabled by issuing the following command:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show rsvp interface</b>
RSVP interface: 1 active
                  Active Subscr- Static      Available   Reserved    Highwater
Interface   State resv   iption  BW          BW          BW          mark
xe-0/2/0.9  Up         1   100%  10Gbps      10Gbps      0bps        0bps                    
</pre>
<p>
After applying the MPLS and RSVP configuration to all the links between the routers, we are done <u>enabling</u> the network to support RSVP-signaled MPLS LSPs. 
We haven’t told any router to actually signal one, so no LSP will form with the present configuration. 
In order to make the routers start signaling the LSPs, we need to manually configure them.
</p>
<p>
After configuring RSVP to run on an interface, any LSP that you want RSVP to signal for is configured in the [protocols mpls] stanza.  In this configuration stanza, we can tell the Tiberius router to try to establish an LSP towards the Commodus router.
</p>            

![RSVP basic configuration](/img/rsvp-the-basic-configuration-4.png "RSVP basic configuration") 
               

<p>
The following configuration command, set protocols mpls label-switched-path simple-lsp-to-Commodus to 1.1.1.4 , is enough to accomplish this. This configuration command, when issued on the Tiberius router, will make the router start using RSVP to create a unidirectional LSP towards the Commodus router. 
</p>
<p>
The LSP created from Tiberius towards Commodus is unidirectional. This means that only the Tiberius router will be able to use this LSP to send packets towards Commodus. Commodus, being the head-end router to this LSP, will not be able to utilize this LSP to send any return traffic towards Tiberius. To enable the Commodus router to send return traffic towards the Tiberius router across an LSP, we need to create one on the Commodus router;
</p>

![RSVP basic configuration](/img/rsvp-the-basic-configuration-5.png "RSVP basic configuration") 

<p>
After the configuration of the LSPs, we need to verify that the configuration is actually working. To see what LSPs are active on the Tiberius router, we can issue the following command:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show mpls lsp</b>
<font color='red'>Ingress</font> LSP: 1 sessions
To              From            State Rt P     ActivePath       LSPname
1.1.1.4         1.1.1.9         Up     0 *                      <font color='red'>simple-lsp-to-Commodus</font>
Total 1 displayed, Up 1, Down 0

<font color='red'>Egress</font> LSP: 1 sessions
To              From            State   Rt Style Labelin Labelout LSPname
1.1.1.9         1.1.1.4         Up       0  1 FF       3        - <font color='red'>simple-lsp-to-Tiberius</font>
Total 1 displayed, Up 1, Down 0

Transit LSP: 0 sessions
Total 0 displayed, Up 0, Down 0                    
</pre>
<p>
This command shows us that there are 2 LSP’s active on the Tiberius router. One LSP from Tiberius towards Commodus and another one from Commodus towards Tiberius. This command is useful as it is giving you a quick overview of what the router is doing LSP-wise. The following command is a lot more revealing and enables us to get a more detailed view of an LSP:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show mpls lsp name simple-lsp-to-Commodus extensive</b>
Ingress LSP: 1 sessions

<font color='red'>1.1.1.4</font>
  From: <font color='red'>1.1.1.9</font>, State: <font color='red'>Up</font>, ActiveRoute: 0, LSPname: <font color='red'>simple-lsp-to-Commodus</font>
  ActivePath:  (primary)
  LSPtype: Static Configured, Penultimate hop popping
  LoadBalance: Random
  Encoding type: Packet, Switching type: Packet, GPID: IPv4
 *Primary                    State: Up
    Priorities: 7 0
    SmartOptimizeTimer: 180
    Computed ERO (S [L] denotes strict [loose] hops): (CSPF metric: 50)
 2.0.0.33 S 2.0.0.38 S 2.0.0.49 S 2.0.0.57 S 2.0.0.54 S
    Received RRO (ProtectionFlag 1=Available 2=InUse 4=B/W 8=Node 10=SoftPreempt 20=Node-ID):
          2.0.0.33 2.0.0.38 2.0.0.49 2.0.0.57 2.0.0.54
   <font color='red'>10 May  5 10:08:24.214 Record Route:  2.0.0.33 2.0.0.38 2.0.0.49 2.0.0.57 2.0.0.54</font>
    9 May  5 10:08:24.214 Up
    8 May  5 10:08:24.171 Originate Call
    7 May  5 10:08:24.168 Clear Call
    6 May  5 10:08:24.168 CSPF: computation result accepted  2.0.0.33 2.0.0.38 2.0.0.49 2.0.0.57 2.0.0.54
    5 May  5 08:24:01.034 Selected as active path
    4 May  5 08:24:01.032 Record Route:  2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.18 2.0.0.54
    3 May  5 08:24:01.032 Up
    2 May  5 08:24:00.980 Originate Call
    1 May  5 08:24:00.980 CSPF: computation result accepted  2.0.0.33 2.0.0.42 2.0.0.66 2.0.0.18 2.0.0.54
  Created: Tue May  5 08:24:00 2015
Total 1 displayed, Up 1, Down 0

Egress LSP: 1 sessions
Total 0 displayed, Up 0, Down 0

Transit LSP: 0 sessions
Total 0 displayed, Up 0, Down 0                    
</pre>

<p>
This command displays a lot more than just the status of the LSP, who the head-end router is and who the tail-end router is. I’ll save some of the output for another post. For now, let’s just look at the Record Route. This list of IP addresses corresponds to the path the LSP is taking through the network. If you’ve taken the trouble to insert PTR-records for all the point-to-point links in your DNS server, you’ll be able to do some lookups to quickly determine what routers the LSP is passing.  This can save you the hassle of logging in to every IP address to determine what router the LSP is passing.
Anyway, we now know that the LSP is up and what route it is taking and we can put it in a picture like this;
</p>

![RSVP basic configuration](/img/rsvp-the-basic-configuration-6.png "RSVP basic configuration") 
    
<p>
    Some additional information can be gather by examining RSVP. Let’s check the Tiberius router for information about the LSP that can be obtained through rsvp show commands.
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show rsvp session name simple-lsp-to-Commodus extensive</b>
Ingress RSVP: 1 sessions

<font color='red'>1.1.1.4</font>
  From: <font color='red'>1.1.1.9</font>, LSPstate: <font color='red'>Up</font>, ActiveRoute: 0
  LSPname: <font color='red'>simple-lsp-to-Commodus</font>, LSPpath: Primary
  LSPtype: Static Configured
  Suggested label received: -, Suggested label sent: -
  Recovery label received: -, Recovery label sent: 299840
  Resv style: 1 FF, Label in: -, Label out: <font color='red'>299840</font>
  Time left:    -, Since: Tue May  5 10:08:24 2015
  Tspec: rate 0bps size 0bps peak Infbps m 20 M 1500
  Port number: sender 2 receiver 9371 protocol 0
  PATH rcvfrom: localclient
  <font color='red'>Adspec: sent MTU 1500</font>
  <font color='red'>Path MTU: received 1500</font>
  PATH sentto: 2.0.0.33 (xe-0/2/0.9) 156 pkts
  RESV rcvfrom: 2.0.0.33 (xe-0/2/0.9) 156 pkts
  Explct route: 2.0.0.33 2.0.0.38 2.0.0.49 2.0.0.57 2.0.0.54
  Record route: &&lt;self> <font color='red'>2.0.0.33 2.0.0.38 2.0.0.49 2.0.0.57 2.0.0.54</font>
Total 1 displayed, Up 1, Down 0

Egress RSVP: 1 sessions
Total 0 displayed, Up 0, Down 0

Transit RSVP: 0 sessions
Total 0 displayed, Up 0, Down 0                    
</pre>
<p>
    This output is similar to the <b>show mpls lsp xxx extensive</b>. 
	It also shows us the tail-end and the head-end router and whether it is up or not. 
	Other things that are interesting is the Adspec, telling us the MTU for the LSP is 1500, and the received label, which is 299840. 
	This label will be installed in the forwarding table for any prefix we decide to route across the LSP.
</p>

<p>
    The creation of the LSP’s enables the network to send traffic across the LSP’s, it will not do so automatically. So with the current configuration, there will be no label-switching going on in our network.
</p>
<p>
    Let’s keep things simple in this scenario and use a static route on both Tiberius and Commodus to send traffic across the LSP’s. Tiberius and Commodus have a second IP address configured under their loopback interfaces, these secondary loopback IP addresses are not advertised in ISIS. In order to create IP connectivity between these IP addresses, we’ll instruct both routers to use their LSP:
</p>				

![RSVP basic configuration](/img/rsvp-the-basic-configuration-7.png "RSVP basic configuration") 
  
<p>
    In the top of the picture above, we can see the configuration command that is necessary to make the Tiberius router send traffic towards the 192.168.1.4/32 prefix across the LSP.
</p>
<p>
    In the bottom of the picture, we can see the configuration needed to make the Commodus router use the LSP to forward traffic across the LSP for the 192.168.1.9/32 prefix.
</p>
<p>
    To verify what the MX is doing with this piece of configuration, we can issue the following commands:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>show route 192.168.1.4</b>

inet.0: 20 destinations, 21 routes (20 active, 0 holddown, 0 hidden)
+ = Active Route, - = Last Active, * = Both

192.168.1.4/32     *[<font color='red'>RSVP/7/1</font>] 04:44:46, metric 50
                    > to 2.0.0.33 via xe-0/2/0.9, label-switched-path <font color='red'>simple-lsp-to-Commodus</font>

play@MX480-TEST:Tiberius> <b>show route forwarding-table matching 192.168.1.4</b>
Logical system: Tiberius
Routing table: default.inet
Internet:
Destination        Type RtRef Next hop           Type Index NhRef Netif
192.168.1.4/32     user     0 2.0.0.33          <font color='red'>Push 299840</font>  1431     2 xe-0/2/0.9
                  
</pre>
<p>
Here, we can see that the label we saw earlier in our 'show rsvp session name simple-lsp-to-Commodus extensive' is installed in the forwarding table.
</p>
<p>
To verify that we have established connectivity between the two routers, we can send a ping from Tiberius towards Commodus, using the 192.168.1.9 address:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>ping 192.168.1.4 source 192.168.1.9 do-not-fragment size 1472</b>
PING 192.168.1.4 (192.168.1.4): 1472 data bytes
1480 bytes from 192.168.1.4: icmp_seq=0 ttl=60 time=1.398 ms
1480 bytes from 192.168.1.4: icmp_seq=1 ttl=60 time=1.402 ms
^C
--- 192.168.1.4 ping statistics ---
2 packets transmitted, 2 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.398/1.400/1.402/0.002 ms

play@MX480-TEST:Tiberius>                    
</pre>
<p>
By default, the Juniper router keeps track of the number of packets and bytes send across an LSP. We can see this using the following command:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Tiberius> <b>clear mpls lsp statistics</b>

play@MX480-TEST:Tiberius> <b>ping 192.168.1.4 source 192.168.1.9 do-not-fragment size 1472 rapid count 1000</b>
PING 192.168.1.4 (192.168.1.4): 1472 data bytes
!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!!
--- 192.168.1.4 ping statistics ---
1000 packets transmitted, 1000 packets received, 0% packet loss
round-trip min/avg/max/stddev = 1.215/1.385/20.605/1.144 ms

play@MX480-TEST:Tiberius> <b>show mpls lsp name simple-lsp-to-Commodus statistics</b>
Ingress LSP: 1 sessions
To              From            State     Packets            Bytes LSPname
1.1.1.4         1.1.1.9         Up           2000          3004000 simple-lsp-to-Commodus
Total 1 displayed, Up 1, Down 0                    
</pre>
<p>
What we have created here are two functional unidirectional RSVP-signaled LSP’s between the Tiberius and the Commodus router. 
By installing a prefix for the LSP on both Tiberius and Commodus, we enabled the routers to send MPLS traffic across the LSP. 
In the end, we verified that there was IP connectivity between two prefixes across the LSP’s. 
In a next post, I’ll take this basic configuration to start doing some more interesting things with RSVP signaled LSP’s. 
I’ll use an RSVP signaled LSP to tunnel LDP and create an l2circuit between Tiberius and Commodus.
</p>
<p>
The complete configuration for this lab is as follows:
</p>			
<pre style="font-size:12px">
Augustus {
    interfaces {
        xe-0/2/0 {
            unit 15 {
                description Caligula;
                vlan-id 15;
                family inet {
                    mtu 1500;
                    address 2.0.0.58/30;
                }
                family iso;
                family mpls;
            }
        }
        xe-0/3/0 {
            unit 12 {
                description Nero;
                vlan-id 12;
                family inet {
                    mtu 1500;
                    address 2.0.0.45/30;
                }
                family iso;
                family mpls;
            }
            unit 13 {
                description Romulus;
                vlan-id 13;
                family inet {
                    mtu 1500;
                    address 2.0.0.49/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 7 {
                family inet {
                    address 1.1.1.7/32;
                }
                family iso {
                    address 49.0010.0010.0100.1007.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/3/0.13 {
                authentication-key "$9$GzjmTzF/0BEHqT39tIR8X7-Yof5F9CuZU/tp0cS"; 
            }
            interface xe-0/2/0.15 {
                authentication-key "$9$ZhD.5Qz6uORik5F/A1IWLxNs4Pfz/9pJG6Atuhc"; 
            }
            interface xe-0/3/0.12 {
                authentication-key "$9$H.Qn/9pRhrP5nCuBSyNdbsaUF39u0IikpB1ReK"; 
            }
        }
        mpls {
            interface xe-0/2/0.15;
            interface xe-0/3/0.12;
            interface xe-0/3/0.13;
        }
        isis {
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.15 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.12 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.13 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.7 {
                level 1 disable;
            }
        }
    }
}
Caligula {
    interfaces {
        xe-0/2/0 {
            unit 5 {
                description Septimus;
                vlan-id 5;
                family inet {
                    mtu 1500;
                    address 2.0.0.18/30;
                }
                family iso;
                family mpls;
            }
        }
        xe-0/3/0 {
            unit 14 {
                description Commodus;
                vlan-id 14;
                family inet {
                    mtu 1500;
                    address 2.0.0.53/30;
                }
                family iso;
                family mpls;
            }
            unit 15 {
                description Augustus;
                vlan-id 15;
                family inet {
                    mtu 1500;
                    address 2.0.0.57/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 8 {
                family inet {
                    address 1.1.1.8/32;
                }
                family iso {
                    address 49.0010.0010.0100.1008.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/2/0.5 {
                authentication-key "$9$j5k5Fn6A1RS.PF/t0hcxNdb4ZQz6tpBDiA0O1rl"; 
            }
            interface xe-0/3/0.14 {
                authentication-key "$9$Yy4UHq.569paZHmTFAtylKM7Vji.TQns25F360O"; 
            }
            interface xe-0/3/0.15 {
                authentication-key "$9$Ox4EIyKMWxwYoEcK87dg4.P5Q/tleW7Nb0BxdVwJZ"; 
            }
        }
        mpls {
            interface xe-0/2/0.5;
            interface xe-0/3/0.14;
            interface xe-0/3/0.15;
        }
        isis {
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.5 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.14 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.15 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.8 {
                level 1 disable;
            }
        }
    }
}
Commodus {
    interfaces {
        xe-0/2/0 {
            unit 14 {
                description Caligula;
                vlan-id 14;
                family inet {
                    mtu 1500;
                    address 2.0.0.54/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 4 {
                family inet {
                    address 1.1.1.4/32 {
                        preferred;
                    }
                    address 192.168.1.4/32;
                }
                family iso {
                    address 49.0010.0010.0100.1004.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/2/0.14 {
                authentication-key "$9$vdz8-wY2aikPX7wgJU.mCtuOhrVb2JZjKMaUDi5T"; 
            }
        }
        mpls {
            label-switched-path simple-lsp-to-Tiberius {
                to 1.1.1.9;
                install 192.168.1.9/32 active;
            }
            interface xe-0/2/0.14;
        }
        isis {
            export isis;
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.14 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.4 {
                level 1 disable;
            }
        }
    }
    policy-options {
        policy-statement isis {
            term deny-192 {
                from {
                    protocol direct;
                    route-filter 192.168.1.4/32 exact;
                }
                then reject;
            }
            term allow-all {
                then accept;
            }
        }
    }
}
Hadrian {
    interfaces {
        xe-0/3/0 {
            unit 9 {
                description Tiberius;
                vlan-id 9;
                family inet {
                    mtu 1500;
                    address 2.0.0.33/30;
                }
                family iso;
                family mpls;
            }
            unit 10 {
                description Romulus;
                vlan-id 10;
                family inet {
                    mtu 1500;
                    address 2.0.0.37/30;
                }
                family iso;
                family mpls;
            }
            unit 11 {
                description Nero;
                vlan-id 11;
                family inet {
                    mtu 1500;
                    address 2.0.0.41/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 10 {
                family inet {
                    address 1.1.1.10/32;
                }
                family iso {
                    address 49.0010.0010.0100.1010.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/3/0.9 {
                authentication-key "$9$XXENs4aJDmfzdb4ZjkTQ0BIElM2gJji.LxDkqm3n"; 
            }
            interface xe-0/3/0.10 {
                authentication-key "$9$oWZHmf5FApBUjmT3/0OKM8XVYq.53nC4aF/9AIR"; 
            }
            interface xe-0/3/0.11 {
                authentication-key "$9$9EGht1hSyKxNbuOhrv8dVUjHqT3REyvMX/CK8LxsY"; 
            }
        }
        mpls {
            interface xe-0/3/0.9;
            interface xe-0/3/0.10;
            interface xe-0/3/0.11;
        }
        isis {
            level 2 {
                authentication-key "$9$jGim5Qz6Au136MXxNY2"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/3/0.9 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.10 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.11 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.10 {
                level 1 disable;
            }
        }
    }
}
Nero {
    interfaces {
        xe-0/2/0 {
            unit 11 {
                description Hadrian;
                vlan-id 11;
                family inet {
                    mtu 1500;
                    address 2.0.0.42/30;
                }
                family iso;
                family mpls;
            }
            unit 12 {
                description Augustus;
                vlan-id 12;
                family inet {
                    mtu 1500;
                    address 2.0.0.46/30;
                }
                family iso;
                family mpls;
            }
        }
        xe-0/3/0 {
            unit 17 {
                description Septimus;
                vlan-id 17;
                family inet {
                    mtu 1500;
                    address 2.0.0.65/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 11 {
                family inet {
                    address 1.1.1.11/32;
                }
                family iso {
                    address 49.0010.0010.0100.1011.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/2/0.11 {
                authentication-key "$9$/oB-ABEcSeX7Vp0EyKW-dGDik5FIRSKvL69eW8Xws"; 
            }
            interface xe-0/2/0.12 {
                authentication-key "$9$V/saUji.z3924UHm56/Ecyl87ZGimPQdb.5TzAt"; 
            }
            interface xe-0/3/0.17 {
                authentication-key "$9$ZfD.5Qz6uORik5F/A1IWLxNs4Pfz/9pJG6Atuhc"; 
            }
        }
        mpls {
            interface xe-0/2/0.11;
            interface xe-0/2/0.12;
            interface xe-0/3/0.17;
        }
        isis {
            level 2 {
                authentication-key "$9$IeARyevMX-b28XkPfT/9"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.11 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/2/0.12 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.17 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.11 {
                level 1 disable;
            }
        }
    }
}
Romulus {
    interfaces {
        xe-0/2/0 {
            unit 10 {
                description Hadrian;
                vlan-id 10;
                family inet {
                    mtu 1500;
                    address 2.0.0.38/30;
                }
                family iso;
                family mpls;
            }
            unit 13 {
                description Augustus;
                vlan-id 13;
                family inet {
                    mtu 1500;
                    address 2.0.0.50/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 6 {
                family inet {
                    address 1.1.1.6/32;
                }
                family iso {
                    address 49.0010.0010.0100.1006.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/2/0.13 {
                authentication-key "$9$Xm/Ns4aJDmfzdb4ZjkTQ0BIElM2gJji.LxDkqm3n"; 
            }
            interface xe-0/2/0.10 {
                authentication-key "$9$1R/ElM8LNYgJcyMX-boaP5QFCuKvL-dsBINbwYGU"; 
            }
        }
        mpls {
            interface xe-0/2/0.10;
            interface xe-0/2/0.13;
        }
        isis {
            level 2 {
                authentication-key "$9$VIbgaZGi.fzDiOREcvM"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.10 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/2/0.13 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.6 {
                level 1 disable;
            }
        }
    }
}
Septimus {
    interfaces {
        xe-0/2/0 {
            unit 17 {
                description Nero;
                vlan-id 17;
                family inet {
                    mtu 1500;
                    address 2.0.0.66/30;
                }
                family iso;
                family mpls;
            }
        }
        xe-0/3/0 {
            unit 5 {
                description Caligula;
                vlan-id 5;
                family inet {
                    mtu 1500;
                    address 2.0.0.17/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 12 {
                family inet {
                    address 1.1.1.12/32;
                }
                family iso {
                    address 49.0010.0010.0100.1012.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/2/0.17 {
                authentication-key "$9$Qg303A0B1hKMX690Icr8LgoJGkPpu1cSeTzhrlK7N"; 
            }
            interface xe-0/3/0.5 {
                authentication-key "$9$Ah-auRSrlMNdsO1SeWXbwjHqmz6hclW87CtMXxN2g"; 
            }
        }
        mpls {
            interface xe-0/2/0.17;
            interface xe-0/3/0.5;
        }
        isis {
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.17 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.5 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.12 {
                level 1 disable;
            }
        }
    }
}
Tiberius {
    interfaces {
        xe-0/2/0 {
            unit 9 {
                description Hadrian;
                vlan-id 9;
                family inet {
                    mtu 1500;
                    address 2.0.0.34/30;
                }
                family iso;
                family mpls;
            }
        }
        lo0 {
            unit 9 {
                family inet {
                    address 1.1.1.9/32 {
                        preferred;
                    }
                    address 192.168.1.9/32;
                }
                family iso {
                    address 49.0010.0010.0100.1009.00;
                }
            }
        }
    }
    protocols {
        rsvp {
            interface xe-0/2/0.9 {
                authentication-key "$9$Md1Lds2gJHqfxNs4ZDmPAp0BclbwgZGivWJDjHTQ"; 
            }
        }
        mpls {
            label-switched-path simple-lsp-to-Commodus {
                to 1.1.1.4;
                install 192.168.1.4/32 active;
            }
            interface xe-0/2/0.9;
        }
        isis {
            export isis;
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; ## SECRET-DATA
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.9 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.9 {
                level 1 disable;
            }
        }
    }
    policy-options {
        policy-statement isis {
            term deny-192 {
                from {
                    protocol direct;
                    route-filter 192.168.1.9/32 exact;
                }
                then reject;
            }
            term allow-all {
                then accept;
            }
        }
    }
}                    
</pre>







