---
layout: post
title: Juniper example class using PyEZ
tags: [automation, juniper, pyez]
image: /img/juniper_logo.jpg
---

Lately, I have been doing a lot of scripting using <b>PyEZ</b> to communicate with the <b>Juniper</b> API. A thing that I have found to be increadibly worthwhile was a move to a more object oriented approach. I have found it to be off great value and wanted to share an example on how you could do this yourself.

Using PyEZ:
===========

When you are using PyEZ, you import the <b>Device</b> class and instantiate an object of your own to work with.

Example:

<pre style="font-size:12px">
from jnpr.junos import Device
from pprint import pprint


with Device(host='10.0.0.1', user='lab', password='lab123', normalize=True) as dev:                                          
    rpc = dev.rpc.get_ospf3_neighbor_information({'format':'json'})
    
pprint(rpc)    
</pre>

As the script grows, it starts to make sense to turn to functions so that it becomes easier to re-use code. I ended up creating a file with most of the commonly used functions that I would import into other scripts whenever I needed them. The following would be something like a function that I would make:


<pre style="font-size:12px">
def get_ospf3_neighbor_ids(dev):
    """
    Gathers information from the &lt;get-ospf3-neighbor-information> RPC.
    
    Returns a dictionary:
        { 
            neighbor-address : { neighbor-id : interface },
        }
    """
    ret = {}
    
    rpc = dev.rpc.get_ospf3_neighbor_information()
    ospf_neighbors = rpc.findall('./ospf3-neighbor')
    
    for neighbor in ospf_neighbors:
        neighbor_address = neighbor.find('./neighbor-address').text
        neighbor_id = neighbor.find('./neighbor-id').text
        interface = neighbor.find('./interface-name').text
        ret[neighbor_address] = { neighbor_id : interface }
    
    return ret
</pre>

I would then import the functions and call them in other scripts, like so:

<pre style="font-size:12px">
dev = Device(host=host, user=username, password=password, normalize=True)
dev.open()    
ospfv2_neighbors = get_ospf_neighbor_ids(dev)
ospfv3_neighbors = get_ospf3_neighbor_ids(dev)
int_dict = get_interface_descriptions(dev)    
dev.close()
</pre>

This worked as it enabled be to re-use a lot of functions and it made scripts a lot more readable. Then someone asked me why I was passing the <b>dev</b> from one funtion to the next. I said this was because setting up the connection takes a few seconds and doing it only once was more efficient. The reply was that I should really be using a class and turn the functions into class methods.


Using a class:
==============

Have a look at the following file <b>juniper_class.py</b>:

<pre style="font-size:12px">
from jnpr.junos import Device

class JunosDevice(Device):
    """Juniper device class for interacting with the device using NETCONF.
    
    """

    def get_ospf_neighbor_ids(self):
        """
        Gathers information from the &lt;get-ospf-neighbor-information> RPC.
        
        Returns a dictionary:
            { 
                neighbor-address : { neighbor-id : interface },
            }
        """      
        ret = {}
        rpc = self.rpc.get_ospf_neighbor_information()
        ospf_neighbors = rpc.findall('./ospf-neighbor')

        for neighbor in ospf_neighbors:
            neighbor_address = neighbor.find('./neighbor-address').text
            neighbor_id = neighbor.find('./neighbor-id').text
            interface = neighbor.find('./interface-name').text
            ret[neighbor_address] = { neighbor_id : interface }

        return ret

    def get_ospf3_neighbor_ids(self):
        """
        Gathers information from the &lt;get-ospf3-neighbor-information> RPC.
        
        Returns a dictionary:
            { 
                neighbor-address : { neighbor-id : interface },
            }
        """     
        ret = {}
       
        rpc = self.rpc.get_ospf3_neighbor_information()
        ospf_neighbors = rpc.findall('./ospf3-neighbor')
        
        for neighbor in ospf_neighbors:
            neighbor_address = neighbor.find('./neighbor-address').text
            neighbor_id = neighbor.find('./neighbor-id').text
            interface = neighbor.find('./interface-name').text
            ret[neighbor_address] = { neighbor_id : interface }
        
        return ret  

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

At the top, right after the import of <b>Device</b>, there is the <b>JunosDevice</b> class. <b>JunosDevice</b> is a child class to <b>Device</b>. One of the things this implies is that the methods available to <b>Device</b> are available to <b>JunosDevice</b> also.

With the creation of additional functions, you basically extend the features that <b>Device</b> has to offer with your own.

In the following file <b>test_juniper_class.py</b>, I am using the <b>JunosDevice</b> as an example:

<pre style="font-size:12px">
from juniper_class import JunosDevice
from pprint import pprint
    
with JunosDevice(host='192.168.118.251', user='lab', password='lab123', normalize=True) as dev:                                  
    pprint(dev.facts)
    print(dev.cli('show version', warning=False))
    pprint(dev.get_ospf_neighbor_ids())
    pprint(dev.get_ospf3_neighbor_ids())
    pprint(dev.get_bgp_summary())   
</pre>



Closing thoughts:
=================

By making your own class inherit the Junipr PyEZ <b>Device</b> class, you equip your own class with everything that PyEZ comes with <b>Device</b> class. After this, you can start extending it with methods that make sense for your environment. 

By putting them in a class, you have something that is very easy to re-use in other scripts and programs. This will make those scripts and programs easier to write because most of the work has allready been done.