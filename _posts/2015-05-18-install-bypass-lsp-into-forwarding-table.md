---
layout: post
title: Installing a bypass LSP into the forwarding table
tags: [juniper, mpls, rsvp]
image: /img/juniper_logo.jpg
---

<p>
After covering link-protection and node-link-protection <a href="https://saidvandeklundert.net/2015-05-17-link-protection-and-node-link-protection-on-juniper-mx/">here</a>, I realized that I forgot one aspect. You can make Junos install the pre-signaled bypass LSP into the forwarding table. This is done by configuring a policy and by applying that policy under the [routing-options forwarding-table export ] stanza.
</p>
<p>
A short example;
</p>

![Installing bypass LSP](/img/install-bypass-lsp-into-forwarding-table.png "Installing bypass LSP")   

<p>
There is an LSP (to_Commodus) that runs from Tiberius to Commodus. The LSP is configured with link-protection and Nero has already signaled for a bypass LSP:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show rsvp session ingress</b>
Ingress RSVP: 2 sessions
To              From            State   Rt Style Labelin Labelout LSPname
1.1.1.12        1.1.1.11        Up       0  1 SE       -   <font color='0099FF'>300896</font> Bypass->2.0.0.66
Total 2 displayed, Up 2, Down 0                    
</pre>
<p>
This bypass LSP is protection the following LSP:
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show mpls lsp transit name to_Commodus</b>
Transit LSP: 7 sessions
To              From            State   Rt Style Labelin Labelout LSPname
1.1.1.4         1.1.1.9         Up       0  1 SE  302160   <font color='red'>300400</font> to_Commodus
Total 1 displayed, Up 1, Down 0                    
</pre>
<p>
When we examine the forwarding table, we can see the transit LSP (300400), but not the bypass LSP (300896):
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show route forwarding-table family mpls</b>
Logical system: Nero
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index NhRef Netif
default            perm     0                    dscd   713     1
0                  user     0                    recv   715     4
1                  user     0                    recv   715     4
2                  user     0                    recv   715     4
13                 user     0                    recv   715     4
301680             user     0 2.0.0.41          Swap 300128  1463     2 xe-0/2/0.11
301712             user     0 2.0.0.45          Pop    1645     2 xe-0/2/0.12
301712(S=0)        user     0 2.0.0.45          Pop    1646     2 xe-0/2/0.12
302016             user     0 2.0.0.41          Swap 300416  1666     2 xe-0/2/0.11
302048             user     0 2.0.0.41          Swap 300448  1441     2 xe-0/2/0.11
302160             user     0 2.0.0.66          Swap <font color='red'>300400</font>  1443     2 xe-0/3/0.17
302176             user     0 2.0.0.66          Swap 300432  1680     2 xe-0/3/0.17
302192             user     0 2.0.0.45          Pop    1682     2 xe-0/2/0.12
302192(S=0)        user     0 2.0.0.45          Pop    1683     2 xe-0/2/0.12                    
</pre>
<p>
The policy to install this bypass LSP into the forwarding table could look like this:
</p>
<pre style="font-size:12px">
set policy-options policy-statement install-bypass term 1 then load-balance per-packet                    
</pre>
<p>
This is a very short policy, but it does the trick. It matches all traffic and it initiate load-balancing.
Also note that even though the policy says per-packet, modern Juniper systems will perform per-flow load-balancing. 
To apply this policy and to make the router install the bypass LSP into the forwarding table, we add the following command:
</p>
<pre style="font-size:12px">
set routing-options forwarding-table export install-bypass                    
</pre>
<p>
Upon examing the forwarding table, we can now see the following;
</p>
<pre style="font-size:12px">
play@MX480-TEST:Nero> <b>show route forwarding-table family mpls</b>
Logical system: Nero
Routing table: default.mpls
MPLS:
Destination        Type RtRef Next hop           Type Index NhRef Netif
default            perm     0                    dscd   713     1
0                  user     0                    recv   715     4
1                  user     0                    recv   715     4
2                  user     0                    recv   715     4
13                 user     0                    recv   715     4
301680             user     0 2.0.0.41          Swap 300128  1463     2 xe-0/2/0.11
301712             user     0 2.0.0.45          Pop    1645     2 xe-0/2/0.12
301712(S=0)        user     0 2.0.0.45          Pop    1646     2 xe-0/2/0.12
302016             user     0 2.0.0.41          Swap 300416  1666     2 xe-0/2/0.11
302048             user     0                    ulst 1048580     2
                              2.0.0.41          Swap 300448  1441     2 xe-0/2/0.11
                              2.0.0.45          Swap 300768  1692     1 xe-0/2/0.12
302160             user     0                    ulst 1048581     2
                              2.0.0.66          Swap <font color='red'>300400</font>  1443     2 xe-0/3/0.17
                              2.0.0.45          Swap <font color='0099FF'>300400</font>, Push <font color='0099FF'>300896</font>(top)  1694     1 xe-0/2/0.12
302176             user     0 2.0.0.66          Swap 300432  1680     2 xe-0/3/0.17
302192             user     0 2.0.0.45          Pop    1682     2 xe-0/2/0.12
302192(S=0)        user     0 2.0.0.45          Pop    1683     2 xe-0/2/0.12                    
</pre>
<p>
The bypass LSP was installed into the forwarding table.
</p>
<p>
Note, I was using an MX480 running 12.3R8.7. I saw that traffic was not being load-balanced. 
The entry in the forwarding table was listing 'ulst' (List of unicast next hops). Normally, this is an indication that traffic is being load-balanced.
The only thing I observed was that the bypass LSP was installed into the forwarding table. 
Only when I interrupted the link between Nero and Septimus did I see the statistics for the bypass LSP increase.
</p>