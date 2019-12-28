---
layout: post
title: Juniper example class using PyEZ
tags: [automation, juniper, pyez]
image: /img/juniper_logo.jpg
---

Recently, a thing that I have found to be increadibly worthwhile was a move to a more object oriented approach. For this reason, I wanted to share an example on how you could do this yourself.

Using PyEZ:
===========

When you are using PyEZ, you import the <b>Device</b> class and instantiate an object of your own to work with:


<pre style="font-size:12px">
from jnpr.junos import Device
from pprint import pprint

with Device(host='10.0.0.1', user='lab', password='lab123', normalize=True) as dev:                                          
    rpc = dev.rpc.get_ospf3_neighbor_information({'format':'json'})
    
pprint(rpc)    
</pre>

As you expand your scripting efforts, it starts to make sense to turn to functions so that it becomes easier to re-use code. Eventually, you might end up creating a file with most of the commonly used functions as I did. I would simply import the file into other scripts and re-use all my previous work like that. 


Using a class:
==============

Having all the functions in one file worked really well and it enabled me to re-use a lot of functions. After a while though, someone pointed out to me that I could just as well extend the <b>Device</b> class with my own specialized <b>subclass</b> that contains the functions (or methods) I use most.

Have a look at the following file <b>juniper_class.py</b>:

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

At the top, right after the import of <b>Device</b>, there is the <b>JunosDevice</b> class. <b>JunosDevice</b> is a child class to <b>Device</b>. One of the things this implies is that the methods available to <b>Device</b> are available to <b>JunosDevice</b> also.

With the creation of additional functions, you basically extend the features that <b>Device</b> has to offer with your own.

In the following file <b>test_juniper_class.py</b>, I am using the <b>JunosDevice</b> as an example:

<pre style="font-size:12px">
from juniper_class import JunosDevice
from pprint import pprint
    
with JunosDevice(host='10.0.19.245', user='lab', password='lab123', normalize=True) as dev: 
    pprint(dev.get_bgp_summary())
</pre>

If we run this script, we see the following:

<pre style="font-size:12px">
<b>sh-4.4# python3 test_class.py</b>
{'10.0.3.48': {'peer-as': '65500',
                'peer-description': 'BGP: gr01.dal',
                'peer-up-time': '2524741'},
 '10.0.3.49': {'peer-as': '65500',
                'peer-description': 'BGP: gr02.dal',
                'peer-up-time': '2523245'},
 '2001:db8:2:5::8': {'peer-as': '65500',
                      'peer-description': 'BGP: ar02.dal',
                      'peer-up-time': '60149802'}}
</pre>


In the example, we instantiated the object and used the <b>get_bgp_summary</b> method to display BGP information. Let's dig a little deeper by opening up an interpretor and connecting to a device:

<pre style="font-size:12px">
sh-4.4# <b>python3</b>          
Python 3.6.8 (default, May 21 2019, 23:51:36) 
[GCC 8.2.1 20180905 (Red Hat 8.2.1-3)] on linux
Type "help", "copyright", "credits" or "license" for more information.
>>> <b>from juniper_class import JunosDevice</b>

>>> <b>dev = JunosDevice(host='192.168.1.1', user='lab', password='lab123', normalize=True)</b>

>>> <b>type(dev)</b>
&lt;class 'juniper_class.JunosDevice'>

>>> <b>dir(dev)</b>
['ON_JUNOS', 'Template', '__class__', '__delattr__', '__dict__', '__dir__', '__doc__', '__enter__', '__eq__', '__exit__', '__format__', '__ge__', '__getattribute__', '__gt__', '__hash__', '__init__', '__init_subclass__', '__le__', '__lt__', '__module__', '__ne__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_auth_password', '_auth_user', '_auto_probe', '_conf_auth_user', '_conf_ssh_private_key_file', '_conn', '_connected', '_fact_style', '_gather_facts', '_hostname', '_j2ldr', '_manages', '_nc_transform', '_norm_transform', '_normalize', '_ofacts', '_port', '_rpc_reply', '_sock_fd', '_ssh_config', '_ssh_private_key_file', '_sshconf_lkup', '_sshconf_path', 'auto_probe', 'bind', 'cli', 'cli_to_rpc_string', 'close', 'connected', 'display_xml_rpc', 'execute', 'facts', 'facts_refresh', '<font color='red'>get_bgp_summary</font>', 'hostname', 'logfile', 'manages', 'master', 'ofacts', 'open', 'password', 'port', 'probe', 're_name', 'rpc', 'timeout', 'transform', 'uptime', 'user']

>>> <b>dev.open()</b>
Device(192.168.1.1)

>>> <b>dev.cli('show version', warning=False)</b>

Hostname: ar01.ams-re0
Model: mx960
Junos: 15.1F4.15-C1.9

&lt;output omitted>

>>> <b>from pprint import pprint</b>
>>> <b>pprint(dev.facts)</b>
{'2RE': True,
 'HOME': '/var/home/admin',
 'RE0': {'last_reboot_reason': 'Router rebooted after a normal shutdown.',
         'mastership_state': 'master',
         'model': 'RE-S-2000',
         'status': 'OK',
         'up_time': '1260 days, 12 hours, 49 minutes, 44 seconds'},

 &lt;output omitted>

>>> <b>dev.close()</b>
>>>
</pre> 

The <b>dev</b> we instantiated shows it is of the &lt;class 'juniper_class.JunosDevice'>. I is a subclass of the Device class and as we can see using <b>dir(dev)</b>, it has all the same methods in addition to the <b>get_bgp_summary</b> extended the class with.

Closing thoughts:
=================

By making your own class inherit the Junipr PyEZ <b>Device</b> class, you equip your own class with everything that PyEZ comes with <b>Device</b> class. After this, you can start extending it with methods that make sense for your environment. 

By putting them in a class, you have something that is very easy to re-use in other scripts and programs. This will make those scripts and programs easier to write because most of the work has allready been done.


