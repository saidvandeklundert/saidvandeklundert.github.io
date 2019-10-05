---
layout: post
title: BIRD BGP filter example
image: /img/bird_logo.png
---
   

![BIRD](/img/bird-filter.png "BIRD")             

<p>
Coming from Junos, I found that manipulating BGP path attributes in BIRD is both straightforward and powerful. I wanted to share an example filter that manipulates the BGP PATH attributes that are changed most often.
</p>
<p>
    I used BIRD as a route-reflector for two MX routers to test the filter. The filter is setup sort of like it can be found on a lot of other platforms. A simple list that results in a terminating action as soon as there is a match. 
</p>

<p>
    The filter:
</p>
<pre style="font-size:12px">
[root@bird ~]# more /etc/bird.filters
filter rr_export {
        # <font color='red'>term 1</font> | set community to 1:100
        if (net = 192.168.1.0/24) then
            {
                bgp_community.add ((1,100));
                accept;
            }
        # <font color='red'>term 2</font> | set community to 1:1000
        if (net = 192.168.2.0/24) then
            {
                bgp_community.add ((1,1000));
                accept;
            }
        # <font color='red'>term 3</font> | set local preference to 175
        if (net = 192.168.3.0/24) then
            {
                bgp_local_pref=175;
                accept;
            }
        # <font color='red'>term 4</font> | prepend an AS
        if (net = 192.168.4.0/24) then
            {
                bgp_path.prepend(65000);
                accept;
            }
        # <font color='red'>term 5</font> | alter the next-hop to whatever (useful for RTBH)
        # make sure BIRD can resolve next-hop, even when used as export filter
        if (net = 192.168.5.0/24) then
            {
                bgp_next_hop = 34.0.1.1;        
                accept;
            }
        # <font color='red'>term 6</font> | alter the MED
        if (net = 192.168.6.0/24) then
            {
                bgp_med  = 175;
                accept;
            }
        # <font color='red'>term 7</font> | match on subnet AND community,
        # then changes two PATH attributes and strip communities
        if (net = 192.168.7.0/24 && (541,541) ~ bgp_community) then
            {
                bgp_community.empty;
                bgp_med  = 241;
                bgp_local_pref=175;
                accept;
            }
        # <font color='red'>term 8</font> | match on subnet or MED
        if (net = 192.168.8.0/24 || bgp_med = 321) then
            {
                bgp_med  = 8;
                bgp_local_pref=8;
                accept;
            }
        # <font color='red'>term 9</font> | match prefix-list style, 
        # example is for any prefix with a mask ranging from 20 - 24
        # Junos: route-filter 0.0.0.0/0 prefix-length-range /20-/24;
        if ( net ~ [ 0.0.0.0/0{20,24} ] ) then
            {
                bgp_med  = 88;
                bgp_local_pref=88;
                accept;
            }
        # end by accepting all other routes
        else accept;
    }
</pre>
                <p>
                    To apply the filter, reference it with the following configuration command:
                </p>
                <pre>
template bgp RR {
..
        export filter rr_export;                                 
..
}</pre>
                <p>
                    Worked for me and I wanted to keep it as an example to copy paste from in the future.
                </p>
                
                <p>
                    <a href="http://bird.network.cz/?get_doc">Chapter 5</a> in the BIRD User's Guide offers some very clear insights into all the things possible with filtering and BIRD.
                </p>
