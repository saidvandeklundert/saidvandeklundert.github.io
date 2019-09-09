---
layout: post
title: NAPALM
image: /img/napalm_logo.png.png
---

Recently, NAPALM and I crossed paths. Up untill this point, most of my scripting efforts involved <b>PyEZ</b>, <b>Paramiko</b> and <b>Netmiko</b>. But after having to work with multiple vendors inside <b>SaltStack</b>, <b>NAPALM</b> started coming up more and more. This post is a short summary of what I have learned so far about NAPALM in general.  

<p align="center">
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_logo.png">
</p>
<br>

NAPALM in a nutshell
====================

Different networking vendors ship their software with their own specific OS and every OS has its own interfaces. By now, most vendors have ensured their software ships with an API. In case you have a multivendor network, one of the challenges you will run into when you want to start automating, is figuring out how all these different APIs work.

Wouldn’t it be nice if there was some sort of library you could import and be done with it? Like a wrapper around all the vendor-specific things.

Enter NAPALM.

Though it does not abstract everything away, it is quite convenient for a lot of basic operations.

<br>

What does NAPALM support?
=========================


At the time of writing, the [docs](https://napalm.readthedocs.io/en/latest/) indicate that NAPALM offers support for the following devices:
- Arista EOS
- Cisco IOS
- Cisco IOS-XR
- Cisco NX-OS
- Juniper JunOS

These devices are managed through the 'core' drivers that are supported and maintained by the people working on NAPALM. 
These drivers are libraries that have been written to deal with the different vendor APIs and they allow NAPALM to deal with different devices. The following table displays the current core libraries:

  
  |                     | EOS       | Junos         | IOS-XR     | NX-OS     | NX-OS SSH  | IOS
  | ------------------- | --------- | ------------- | ---------- | --------- | ---------- | ---- 
  | **Driver Name**     | eos       | junos         | iosxr      | nxos      | nxos_ssh   | ios
  | **Backend library** | [pyeapi](https://github.com/arista-eosplus/pyeapi)  | [junos-eznc](https://github.com/Juniper/py-junos-eznc)  | [pyIOSXR](https://github.com/fooelisa/pyiosxr)  | [pynxos](https://github.com/networktocode/pynxos)  | [netmiko](https://github.com/ktbyers/netmiko)  | [netmiko](https://github.com/ktbyers/netmiko)
  

You can also use these backend libraries in your scripts. Two backend libraries were actually pretty familiar to me already. I worked with `junos-eznc`, a.k.a. PyEZ, when managing Juniper devices. And `netmiko` was familiar because of some scripting I did against several different vendors. The `netmiko` library, when used outside of NAPALM as standalone library, is a ```Multi-vendor library to simplify Paramiko SSH connections to network devices```. In NAPALM, it is used to get NAPALM to talk to NX-OS over SSH and IOS (which does not have an API).

In addition to the core drivers, there are also various 'community drivers'. Community drivers are maintained under their own repository and can be used by NAPALM. I found a list of community drivers [here](https://github.com/napalm-automation-community).

<br>

NAPALM in a Python script
=========================

NAPALM provides you with several basic functions that you can use to interact with different vendors. Let's look at an example where we retrieve information from a Juniper and a Cisco and use NAPALM to do the following:
- display information that describes the device
- display information about the BGP neighbors

Doing this using NAPALM is very straightforward.

<p align="center" >
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_information_example.png">
</p>

Pretty much, the only thing we need to do is displayed above. First, we need to select the proper driver. After that, we call the `getters` we are interested in. In this case, we are going for `get_facts` and `get_bgp_neighbors`.

The Python required to perform the above when connection to a Juniper device will be something like the following:

```python
import napalm
from pprint import pprint as pp
driver = napalm.get_network_driver('junos')
device = driver(hostname='169.50.169.100', username='salt', password='salt123')
device.open()
pp(device.get_facts())
pp(device.get_bgp_neighbors())
device.close()
```

Notice how we only see NAPALM references in this Python and no PyEZ. 

When connecting to a Juniper device using NAPALM, you need to use the `junos` driver. This will in turn reference the PyEZ backend library (`junos-eznc`).This library uses NETCONF to communicate with the Juniper XML API. 

When we run this example in the interpreter, we get the following:

```python
bash-4.4$ python
Python 2.7.16 (default, May  6 2019, 19:35:26) 
[GCC 8.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import napalm                  
>>> from pprint import pprint as pp
>>> driver = napalm.get_network_driver('junos')
>>> device = driver(hostname='169.50.169.171', username='salt-r6', password='salt123')
>>> device.open()
>>> pp(device.get_facts())
{u'fqdn': u'vmx01',
 u'hostname': u'vmx01',
 u'interface_list': ['ge-0/0/1', '.local.', 'lo0', 'lsi'],
 u'model': u'VMX',
 u'os_version': u'19.1R1.6',
 u'serial_number': u'VM5CB1E69517',
 u'uptime': 12351159,
 u'vendor': u'Juniper'}
>>> pp(device.get_bgp_neighbors())
{u'global': {u'peers': {u'10.0.0.14': {u'address_family': {u'ipv4': {u'accepted_prefixes': -1,
                                                                     u'received_prefixes': -1,
                                                                     u'sent_prefixes': -1},
                                                           u'ipv6': {u'accepted_prefixes': -1,
                                                                     u'received_prefixes': -1,
                                                                     u'sent_prefixes': -1}},
                                       u'description': u'',
                                       u'is_enabled': True,
                                       u'is_up': True,
                                       u'local_as': 1,
                                       u'remote_as': 1,
                                       u'remote_id': u'10.0.0.14',
                                       u'uptime': 885023},
                        '10.0.0.15': {u'address_family': {u'ipv4': {u'accepted_prefixes': -1,
                                                                    u'received_prefixes': -1,
                                                                    u'sent_prefixes': -1},
                                                          u'ipv6': {u'accepted_prefixes': -1,
                                                                    u'received_prefixes': -1,
                                                                    u'sent_prefixes': -1}},
                                      u'description': u'',
                                      u'is_enabled': True,
                                      u'is_up': True,
                                      u'local_as': 1,
                                      u'remote_as': 1,
                                      u'remote_id': u'10.0.0.15',
                                      u'uptime': 789397}},
             u'router_id': u'10.0.0.6'}}
>>> device.close()
>>> 
```

If we were to check the log on the device during the time where we gathered the BGP neighbor information, we can see all the RPCs our NAPALM script called:

```
Sep  9 13:20:50  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-instance-information'
Sep  9 13:20:50  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-bgp-neighbor-information instance=master'
Sep  9 13:20:50  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-bgp-summary-information instance=master'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-bgp-neighbor-information instance=cust-1'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-bgp-summary-information instance=cust-1'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-bgp-neighbor-information instance=mgmt_junos'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt-r6' used NETCONF client to run command 'get-bgp-summary-information instance=mgmt_junos'
```
<br>

The beauty here is that you do not need to know all the specifics with regards to the Juniper API. Let's change the example code to something that works for IOS XR:

```python
import napalm
from pprint import pprint as pp
driver = napalm.get_network_driver('iosxr')
device = driver(hostname='169.50.169.101', username='salt', password='salt123')
device.open()
pp(device.get_facts())
pp(device.get_bgp_neighbors())
device.close()
```

The only thing we changed is the value of the driver!!

When we run this in the interpretor, we get the following:

```python
bash-4.4$ python
Python 2.7.16 (default, May  6 2019, 19:35:26) 
[GCC 8.3.0] on linux2
Type "help", "copyright", "credits" or "license" for more information.
>>> import napalm
>>> from pprint import pprint as pp
>>> driver = napalm.get_network_driver('iosxr')
>>> device = driver(hostname='169.50.169.170', username='salt', password='salt123')
>>> device.open()
pp(device.get_facts())
>>> pp(device.get_facts())

{u'fqdn': u'ios_xr_1',
 u'hostname': u'ios_xr_1',
 u'interface_list': [u'GigabitEthernet0/0/0/0',
                     u'GigabitEthernet0/0/0/0.1156',
                     u'GigabitEthernet0/0/0/1',
                     u'GigabitEthernet0/0/0/1.10',
                     u'GigabitEthernet0/0/0/2',
                     u'GigabitEthernet0/0/0/2.12',
                     u'GigabitEthernet0/0/0/3',
                     u'GigabitEthernet0/0/0/3.2002',
                     u'Loopback0',
                     u'MgmtEth0/RP0/CPU0/0',
                     u'Null0'],
 u'model': u'R-IOSXRV9000-CC',
 u'os_version': u'6.6.2',
 u'serial_number': u'9E478CB8391',
 u'uptime': 20724,
 u'vendor': u'Cisco'}
>>> 
>>> pp(device.get_bgp_neighbors())
{u'cust-1': {u'peers': {}, u'router_id': u'10.0.1.1'},
 u'global': {u'peers': {u'10.0.0.14': {u'address_family': {u'ipv4': {u'accepted_prefixes': 6,
                                                                     u'received_prefixes': 6,
                                                                     u'sent_prefixes': 2}},
                                       u'description': u'',
                                       u'is_enabled': False,
                                       u'is_up': True,
                                       u'local_as': 1,
                                       u'remote_as': 1,
                                       u'remote_id': u'10.0.0.14',
                                       u'uptime': 20573},
                        u'10.0.0.15': {u'address_family': {u'ipv4': {u'accepted_prefixes': 6,
                                                                     u'received_prefixes': 6,
                                                                     u'sent_prefixes': 2}},
                                       u'description': u'',
                                       u'is_enabled': False,
                                       u'is_up': True,
                                       u'local_as': 1,
                                       u'remote_as': 1,
                                       u'remote_id': u'10.0.0.15',
                                       u'uptime': 20573}},
             u'router_id': u'10.0.1.1'}}
>>> device.close()
>>> 
```

Notice how we the only thing we needed to change was the driver. We changed it from `junos` to `iosxr` and after that, we were good to go.

In the example, I used `get_bgp_neighbors` that NAPALM provides us with. There are many more [NAPALM getters](https://napalm.readthedocs.io/en/latest/support/index.html#getters-support-matrix) for you to check out though.

NAPALM outside of a Python script
=================================


As it says on the [NAPALM website](https://napalm-automation.net/), `Napalm plays nice with others!`. Efforts have been made to enable the use of NAPALM outside of a Python script. You can use it for Ansible, Stackstorm and SaltStack. In the case of SaltStack, the automation framework I am most familiar with, this was done through the creation of a proxy minion and several modules that exposes most of the methods available in NAPALM.





Closing
============

my connection
- SaltStack
- IOS XR
- Junos
- Arista






Supported devices:
https://napalm.readthedocs.io/en/latest/support/index.html

IOS caveats:
https://napalm.readthedocs.io/en/latest/support/ios.html

