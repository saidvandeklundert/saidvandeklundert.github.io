---
layout: post
title: Link-protection and node-link-protection on Juniper MX - complete configuration
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---


<p>
This is the complete configuration used in <a href="2015-05-17-link-protection-and-node-link-protection-on-juniper-mx.md" >Link-protection and node-link-protection on Juniper MX</a> and several other articles. The routers in the topology drawing are all logical systems.
</p>
<p>
The routers are running IS-IS. All interfaces are MPLS and RSVP-enabled.
Under the RSVP configuration, there are also the ‘aggregate’ and the ‘reliable’ statement. These were not discussed and they are not necessary for link-protection or node-link-protection. These two statements simply reduce RSVP refresh and enable reliable delivery.
</p>
<p>
There is also an L2circuit defined (and working) between Commodus and Tiberius.
</p>

![RSVP node-link protection](/img/node-link-protection-1.png "RSVP node-link protection")   

<pre style="font-size:12px">
play@MX480-TEST> <b>show configuration logical-systems</b>
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/2/0.15 {
                authentication-key "$9$ZhD.5Qz6uORik5F/A1IWLxNs4Pfz/9pJG6Atuhc"; 
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.12 {
                authentication-key "$9$H.Qn/9pRhrP5nCuBSyNdbsaUF39u0IikpB1ReK"; 
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            interface xe-0/2/0.15;
            interface xe-0/3/0.12;
            interface xe-0/3/0.13;
        }
        isis {
            traffic-engineering {
                family inet {
                    shortcuts;
                }
            }
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; 
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.14 {
                authentication-key "$9$Yy4UHq.569paZHmTFAtylKM7Vji.TQns25F360O"; 
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.15 {
                authentication-key "$9$Ox4EIyKMWxwYoEcK87dg4.P5Q/tleW7Nb0BxdVwJZ";
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            interface xe-0/2/0.5;
            interface xe-0/3/0.14;
            interface xe-0/3/0.15;
        }
        isis {
            traffic-engineering {
                family inet {
                    shortcuts;
                }
            }
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; 
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
        xe-0/3/0 {
            unit 6 {
                description Septimus;
                vlan-id 6;
                family inet {
                    mtu 1500;
                    address 2.0.0.21/30;
                }
                family iso;
                family mpls;
            }
            unit 3556 {
                description Mars;
                encapsulation vlan-ccc;
                vlan-id 3556;
            }
        }
        lo0 {
            unit 4 {
                family inet {
                    address 1.1.1.4/32 {
                        preferred;
                    }
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.6 {
                authentication-key "$9$1R/ElM8LNYgJcyMX-boaP5QFCuKvL-dsBINbwYGU"; 
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            revert-timer 120;
            label-switched-path to_Tiberius {
                to 1.1.1.9;
                ldp-tunneling;
                node-link-protection;
                primary via_Caligula;
                secondary via_Septimus {
                    standby;
                }
            }
            path via_Caligula {
                1.1.1.8 strict;
            }
            path via_Septimus {
                1.1.1.12 strict;
            }
            interface xe-0/2/0.14;
            interface xe-0/3/0.6;
        }
        isis {
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; 
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.14 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/3/0.6 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.4 {
                level 1 disable;
            }
        }
        ldp {
            interface lo0.4;
            session 1.1.1.9 {
                authentication-key "$9$87Z7dsiHmznCikfz36u0LxN-Yo"; 
            }
        }
        l2circuit {
            neighbor 1.1.1.9 {
                interface xe-0/3/0.3556 {
                    virtual-circuit-id 3556;
                }
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.10 {
                authentication-key "$9$oWZHmf5FApBUjmT3/0OKM8XVYq.53nC4aF/9AIR"; 
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.11 {
                authentication-key "$9$9EGht1hSyKxNbuOhrv8dVUjHqT3REyvMX/CK8LxsY";
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            interface xe-0/3/0.9;
            interface xe-0/3/0.10;
            interface xe-0/3/0.11;
        }
        isis {
            traffic-engineering {
                family inet {
                    shortcuts;
                }
            }
            level 2 {
                authentication-key "$9$jGim5Qz6Au136MXxNY2"; 
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/2/0.12 {
                authentication-key "$9$V/saUji.z3924UHm56/Ecyl87ZGimPQdb.5TzAt"; 
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.17 {
                authentication-key "$9$ZfD.5Qz6uORik5F/A1IWLxNs4Pfz/9pJG6Atuhc";
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            interface xe-0/2/0.11;
            interface xe-0/2/0.12;
            interface xe-0/3/0.17;
        }
        isis {
            traffic-engineering {
                family inet {
                    shortcuts;
                }
            }
            level 2 {
                authentication-key "$9$IeARyevMX-b28XkPfT/9"; 
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
        xe-0/3/0 {
            unit 3 {
                description Tiberius;
                vlan-id 3;
                family inet {
                    mtu 1500;
                    address 2.0.0.9/30;
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/2/0.10 {
                authentication-key "$9$1R/ElM8LNYgJcyMX-boaP5QFCuKvL-dsBINbwYGU"; 
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.3 {
                authentication-key "$9$1R/ElM8LNYgJcyMX-boaP5QFCuKvL-dsBINbwYGU"; 
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            interface xe-0/2/0.10;
            interface xe-0/2/0.13;
            interface xe-0/3/0.3;
        }
        isis {
            traffic-engineering {
                family inet {
                    shortcuts;
                }
            }
            level 2 {
                authentication-key "$9$VIbgaZGi.fzDiOREcvM"; 
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
            interface xe-0/3/0.3 {
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
            unit 6 {
                description Commodus;
                vlan-id 6;
                family inet {
                    mtu 1500;
                    address 2.0.0.22/30;
                }
                family iso;
                family mpls;
            }
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/3/0.5 {
                authentication-key "$9$Ah-auRSrlMNdsO1SeWXbwjHqmz6hclW87CtMXxN2g"; 
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/2/0.6 {
                authentication-key "$9$1R/ElM8LNYgJcyMX-boaP5QFCuKvL-dsBINbwYGU"; 
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            interface xe-0/2/0.17;
            interface xe-0/3/0.5;
            interface xe-0/2/0.6;
        }
        isis {
            traffic-engineering {
                family inet {
                    shortcuts;
                }
            }
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; 
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.6 {
                point-to-point;
                level 1 disable;
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
            unit 3 {
                description Romulus;
                vlan-id 3;
                family inet {
                    mtu 1500;
                    address 2.0.0.10/30;
                }
                family iso;
                family mpls;
            }
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
            unit 3556 {
                description Sol;
                encapsulation vlan-ccc;
                vlan-id 3556;
            }
        }
        lo0 {
            unit 9 {
                family inet {
                    address 1.1.1.9/32 {
                        preferred;
                    }
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
                aggregate;
                reliable;
                link-protection;
            }
            interface xe-0/2/0.3 {
                authentication-key "$9$1R/ElM8LNYgJcyMX-boaP5QFCuKvL-dsBINbwYGU"; 
                aggregate;
                reliable;
                link-protection;
            }
        }
        mpls {
            revert-timer 0;
            label-switched-path to_Commodus {
                to 1.1.1.4;
                ldp-tunneling;
                link-protection;
                primary via_Hadrian;
                secondary via_Romulus {
                    standby;
                }
            }
            path via_Romulus {
                1.1.1.6 strict;
            }
            path via_Hadrian {
                1.1.1.10 strict;
            }
            interface xe-0/2/0.9;
            interface xe-0/2/0.3;
        }
        isis {
            level 2 {
                authentication-key "$9$0Uz1OESrlMXNbKMDkqmF3"; 
                authentication-type md5;
                wide-metrics-only;
            }
            interface xe-0/2/0.3 {
                point-to-point;
                level 1 disable;
            }
            interface xe-0/2/0.9 {
                point-to-point;
                level 1 disable;
            }
            interface lo0.9 {
                level 1 disable;
            }
        }
        ldp {
            interface lo0.9;
            session 1.1.1.4 {
                authentication-key "$9$RKySlMsYoDHms2JDikTQEcyrWx"; 
            }
        }
        l2circuit {
            neighbor 1.1.1.4 {
                interface xe-0/2/0.3556 {
                    virtual-circuit-id 3556;
                }
            }
        }
    }
}

                    
</pre>