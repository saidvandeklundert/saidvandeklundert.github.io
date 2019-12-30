---
layout: post
title: Juniper example class using PyEZ
tags: [automation, juniper, pyez]
image: /img/juniper_logo.jpg
---

Extending the Junos PyEZ <b>Device</b> class with my own subclass helped me to more easily re-use my code and it made my scripts less of a clutter. This article is a short example on how to do this.

Extending the base Device class:
================================

When you are using PyEZ, you import the <b>Device</b> class and instantiate an object to work with. The following is a pretty straightforward example:

<pre style="font-size:12px">
from jnpr.junos import Device

with <b>Device</b>(host='router_1', user='lab', password='lab123', normalize=True) as router_1:                                          
    rpc = router_1.rpc.get_bgp_summary_information()
</pre>

In the previous example, the <b>Device</b> class is used to create an object named 'router_1'. Using the <b>rpc</b> method, the BGP summary information is retrieved from the device. Because we open the connection to the device using a context manager (<b>with</b>), the connection to the device is automatically opened and closed.

As you expand your scripting efforts, using functions will start to make sense. This will make it easier to re-use code. You could put your functions in a single file and import them in other scripts whenever you require them. 

Another thing you could consider is to extend the <b>Device</b> class with your own <b>subclass</b>. Have a look at the following example <b>juniper_class.py</b>:

<pre style="font-size:12px">
from jnpr.junos import Device

class JunosDevice(Device):

    def get_bgp_summary(self):
        pass     
</pre>

After the import of <b>Device</b>, the <b>JunosDevice</b> class is defined. The parent class, <b>Device</b>, is passed to it as a parameter. 

Because <b>JunosDevice</b> is a child class to <b>Device</b>, all the methods that are available to <b>Device</b> are available to <b>JunosDevice</b> also. You do not have to declare anything to gain access to <b>rpc</b>, <b>cli</b>, <b>facts</b>, etc. 

We can also see that the child class is extended with a method called <b>get_bgp_summary</b>. This is something that you can do in addition to what <b>Device</b> has to offer. Instead of putting in a <b>pass</b>, let's make the <b>get_bgp_summary</b> function do something:

<pre style="font-size:12px">
from jnpr.junos import Device

class JunosDevice(Device):

    def get_bgp_summary(self):
        """        
        Gathers information from the &lt;get-bgp-summary-information> RPC.
        
        Returns a dictionary:
            { 
                peer-address : {
                    'peer-as' : peer_as,
                    'peer-description' : peer_description,
                    'peer-up-time' : peer_up_time,
                }
            }        
        """
        ret = {}
       
        rpc = self.rpc.get_bgp_summary_information()
        bgp_peers = rpc.findall('.//bgp-peer')
        
        for peer in bgp_peers:
            peer_address = peer.find('./peer-address').text
            peer_as = peer.find('./peer-as').text            
            peer_up_time = peer.find('./elapsed-time').attrib['seconds']
            
            peer_description = None
            if peer.find('./description') is not None:
                peer_description = peer.find('./description').text
            
            ret[peer_address] = {                 
                'peer-as' : peer_as,
                'peer-description' : peer_description,
                'peer-up-time' : peer_up_time,
            }

        return ret     
</pre>

The subclass now has an extra method that can 'dictify' the BGP summary information that we think is interesting with the key / values that we like to work with. Working with the <b>JunosDevice</b> class is no different then working with the <b>Device</b> class. The following <b>test_juniper_class.py</b> is an example on how you could use it:

<pre style="font-size:12px">
from juniper_class import JunosDevice
from pprint import pprint
    
with JunosDevice(host='router_1', user='lab', password='lab123', normalize=True) as router_1: 
    pprint(router_1.get_bgp_summary())
</pre>

We import the <b>JunosDevice</b> class and use it to setup a connection with router_1. After this, we call the newly created <b>get_bgp_summary()</b> method which will give us the following:

<pre style="font-size:12px">
<b>sh-4.4# python3 test_class.py</b>
{'10.0.3.48': {'peer-as': '65500',
                'peer-description': 'gr01.dal',
                'peer-up-time': '2524741'},
 '10.0.3.49': {'peer-as': '65500',
                'peer-description': 'gr02.dal',
                'peer-up-time': '2523245'},
 '2001:db8:2:5::8': {'peer-as': '65500',
                      'peer-description': 'ar02.dal',
                      'peer-up-time': '60149802'}}
</pre>


Closing thoughts:
=================

By making your own class inherit the Juniper PyEZ Device class, it will be equipped with everything that PyEZ comes with. And by extending it with methods that make sense for your environment, you will have something that is very easy to re-use, making future scripts and programs easier to write and maintain.


