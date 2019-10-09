---
layout: post
title: Juniper QFX and storm control
tags: [juniper, qfx ]
image: /img/juniper_logo.jpg
---

                  
<p>
    Recently, I deployed a VCF consisting of some QFX5100's and some EX4300's. 
    I found that the default configuration did not really protect the network well enough and I thought I’d share it in this post.
</p>                    
<p>    
    On the QFX, you’ll find that storm-control is enabled by default. 
    The first thing you'll notice is that the hierarchy under which the configuration is found has been moved from the [ ethernet-switching-options ] to the [forwarding-options ]. 
    Another change besides the change in hierarchy, is that storm-control is now configured in two steps. The first step is the configuration of a storm-control-profile. 
    The second step is applying this profile to the interfaces.
</p>
<p>
    On the QFX, the default profile configuration is as follows:
</p>
<pre style="font-size:12px">
play@VCF# show forwarding-options
storm-control-profiles default {
    all;
}
</pre>                 
<p>
    This profile will set storm-control at 80% with the default action, which is to drop frames. 
    Straight out of the box, this default storm-control profile is applied as follows:
</p>
<pre style="font-size:12px">
xe-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members default;
            }
            storm-control default;
        }
    }
}
</pre>                                   
<p>
    This default configuration was not very satisfying to me. 
    With storm-control enabled at 80%, you might as well leave it turned off.
    Additionally to this, I did not want to see thousands of lines of storm-control configuration. To this end, I configured the following:
</p>
<pre style="font-size:12px">
set forwarding-options storm-control-profiles storm-control-5m all bandwidth-level 5000
</pre>                    
<p>
    This creates a storm-control profile that will be triggered at 5Mbit. 
</p>
<pre style="font-size:12px">
set groups vcf-interface interfaces &lt;ge-*&gt; unit 0 family ethernet-switching storm-control storm-control-5m
set groups vcf-interface interfaces &lt;xe-*&gt; unit 0 family ethernet-switching storm-control storm-control-5m
set groups vcf-interface interfaces &lt;et-*&gt; unit 0 family ethernet-switching storm-control storm-control-5m
</pre>                  
<p>
    This creates a group that applies the storm control profile to all possible interfaces on the switch.
</p>
<pre style="font-size:12px">
set interfaces apply-groups vcf-interface
</pre>                  
<p>
    This last statement will actually apply the storm control profile to all the interfaces. With these statements applied, you will be able to see the following in the configuration:
</p>
<pre style="font-size:12px">
xe-0/0/0 {
    unit 0 {
        family ethernet-switching {
            vlan {
                members default;
        }
    }
}
</pre>    
<p>
    And now with diplay inheritance:
</p>
<pre style="font-size:12px">
play@VCF> show configuration interfaces xe-0/0/0 | display inheritance
unit 0 {
    family ethernet-switching {
        vlan {
            members default;
        }
        ##
        ## 'storm-control' was inherited from group 'vcf-interface'
        ## 'storm-control-5m' was inherited from group 'vcf-interface'
        ##
        storm-control storm-control-5m;
    }
}
</pre>     
<p>
    If there are individual links for which another storm control profile is desired, simply configure one and apply it directly under the interface. The interface configuration will take precedence over the 'apply groups' configuration.
</p>
<p>
    In my opinion, this configuration is more safe and concise. And by having storm control go into effect at 5Mbit versus 8Gbit I feel safer already. 
</p> 