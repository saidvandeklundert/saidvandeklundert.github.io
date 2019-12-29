---
layout: post
title: Juniper example class using PyEZ
tags: [automation, juniper, pyez]
image: /img/juniper_logo.jpg
---

A recent move to a more object oriented approach has been increadibly worthwhile to me. In this article I want to share an example on how you can extend the base Junos PyEZ <b>Device class</b>.

Using a class:
==============

When you are using PyEZ, you import the <b>Device</b> class and instantiate an object of your own to work with:

<pre style="font-size:12px">
from jnpr.junos import Device

with Device(host='10.0.0.1', user='lab', password='lab123', normalize=True) as dev:                                          
    rpc = dev.rpc.get_bgp_summary_information()
    ... 
</pre>

As you expand your scripting efforts, it starts to make sense to turn to functions so that it becomes easier to re-use code. Eventually, you might end up creating a file with most of the commonly used functions as I did. Most often, I would import the functions into other scripts and re-use all my previous work like that. 

Having all the functions in one file worked really well and it enabled me to re-use a lot of functions. After a while though, someone pointed out to me that I could just as well extend the <b>Device</b> class with my own <b>subclass</b> that contains the functions I need.

Have a look at the following example <b>juniper_class.py</b>:

<pre style="font-size:12px">
from jnpr.junos import Device

class JunosDevice(Device):
    """Juniper subclass of Device.    
    """

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

After the import of <b>Device</b>, there is the <b>JunosDevice</b> class. <b>JunosDevice</b> is a child class to <b>Device</b>. One of the things this implies is that the methods available to <b>Device</b> are available to <b>JunosDevice</b> also. With the creation of additional functions, we extend the features that <b>Device</b> has to offer with our own.

The following <b>test_juniper_class.py</b> is an example on how to use the <b>JunosDevice</b> class:

<pre style="font-size:12px">
from juniper_class import JunosDevice
from pprint import pprint
    
with JunosDevice(host='10.0.19.245', user='lab', password='lab123', normalize=True) as dev: 
    pprint(dev.get_bgp_summary())
</pre>

We import the <b>JunosDevice</b> class and use it to setup a connection with a device. After this, we call the newly created <b>get_bgp_summary()</b> method. We get the following when we run the script:

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

By making your own class inherit the Junipr PyEZ <b>Device</b> class, you equip your own class with everything that PyEZ comes with <b>Device</b> class. After this, you can start extending it with methods that make sense for your environment. 

By putting them in a class, you have something that is very easy to re-use in other scripts and programs. This will make those scripts and programs easier to write because most of the work has allready been done.


