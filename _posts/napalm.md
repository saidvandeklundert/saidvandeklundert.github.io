---
layout: post
title: NAPALM
image: /img/napalm_logo.png.png
---

Recently, I have become a bit more interested in NAPALM. Up untill this point, most of my scripting efforts involved <b>PyEZ</b>, <b>paramiko</b> and <b>Netmiko</b>. But after having to work with multiple vendors inside <b>SaltStack</b>, <b>NAPALM</b> started coming up more and more. This post is a short summary of what I have learned so far about NAPALM in general.  

<p align="center">
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_logo.png">
</p>
<br>

NAPALM in a nutshell
====================

Different networking vendors ship their boxes/software with their own specific OS and every OS has its own interfaces. By now, most vendors have ensured their software ships with an API. In case you have a multivendor network, one of the challenges you will run into when you want to start automating, is figuring out how all these different APIs work.

Wouldn’t it be nice if there was some sort of library you could import and be done with it? Like a wrapper around all the vendor-specific things.

Enter NAPALM.

Though it does not abstract everything away, it is quite convenient for a lot of basic operations.

<br>

NAPALM, Network Automation and Programmability Abstraction Layer with Multivendor support
=========================================================================================


At the time of writing, the docs ( https://napalm.readthedocs.io/en/latest/ ) indicate the NAPALM offers support for the following devices:
- Arista EOS
- Cisco IOS
- Cisco IOS-XR
- Cisco NX-OS
- Juniper JunOS

These 'core' drivers are supported and maintained by the people working on NAPALM. In addition to the core drivers, there are also some 'community drivers'. These are maintained under their own repository and can be used by NAPALM.

The core drivers in NAPALM allow it to deal with the different devices. These drivers are libraries written to deal with the different vendor APIs. The following picture shows the current core libraries:

  
  |                     | EOS       | Junos         | IOS-XR     | NX-OS     | NX-OS SSH  | IOS
  | ------------------- | --------- | ------------- | ---------- | --------- | ---------- | ---- 
  | **Driver Name**     | eos       | junos         | iosxr      | nxos      | nxos_ssh   | ios
  | **Backend library** | `pyeapi`_ | `junos-eznc`_ | `pyIOSXR`_ | `pynxos`_ | `netmiko`_ | `netmiko`_
  

.. _pyeapi: https://github.com/arista-eosplus/pyeapi
.. _junos-eznc: https://github.com/Juniper/py-junos-eznc
.. _pyIOSXR: https://github.com/fooelisa/pyiosxr
.. _pynxos: https://github.com/networktocode/pynxos
.. _netmiko: https://github.com/ktbyers/netmiko

As explained earlier, these backend libraries are used by NAPALM to communicate with the different vendors. The two backend libraries that stood out to me were `junos-eznc` and `netmiko`. The first one is familiar through earlier work with PyEZ. The latter was familiar for some scripting I did against several other vendors. The `netmiko` library, when used outside of NAPALM as standalone library, is a `Multi-vendor library to simplify Paramiko SSH connections to network devices`. To enable NAPALM to work with NX-OS over SSH and IOS (which does not have an API), they opted to use this as a backend library.

<br>

NAPALM in a Python script
=========================

So when you use NAPALM in a Python script, what exactly happens? Since I am most familiar with Juniper, I thought I'd explain it from a Juniper point of view. Let's use NAPALM to do the following:
- connect to a Juniper device
- display information that describes the device
- display information about the BGP neighbors
- close the connection to the device

The python required to perform the above is the following:

```python
import napalm
from pprint import pprint as pp
driver = napalm.get_network_driver('junos')
device = driver(hostname='169.50.169.171', username='salt-r6', password='salt123')
device.open()
pp(device.get_facts())
pp(device.get_bgp_neighbors())
device.close()
```

Only NAPALM, no PyEZ in the script. 

<p align="center" >
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_talking_to_junos.png">
</p>

When you talk to a Juniper device, you tell NAPALM to use the `junos` driver, which will in turn reference to the PyEZ backend library (`junos-eznc`).This library uses NETCONF to communicate with the Juniper XML API. This API is an interface of `MGD` that exposes `Junos` and allows you to pass instructions or retreive information.

When we run this in the interpreter, we get the following:

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
device = driver(hostname='169.50.169.170', username='salt', password='salt123')
device.open()
pp(device.get_facts())
pp(device.get_bgp_neighbors())
device.close()
```

The only thing we changed is the value of the driver and running this in the interpretor would give the following:

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

NAPALM outside of a Python script
=================================


As it says on the NAPALM website (https://napalm-automation.net/ ), `Napalm plays nice with others!`. Efforts have been made to enable the use of NAPALM outside of a Python script. You can use it for Ansible, Stackstorm and SaltStack. In the case of SaltStack, the automation framework I am most familiar with, this was done through the creation of a proxy minion and several modules that exposes most of the methods available in NAPALM.











Supported devices:
https://napalm.readthedocs.io/en/latest/support/index.html

IOS caveats:
https://napalm.readthedocs.io/en/latest/support/ios.html




In this post,

main link: https://github.com/napalm-automation/napalm





NAPALM:
- Napalm is a vendor neutral, cross-platform open source project that provides a unified API to network devices.
- Napalm plays nice with others! Ansible, SaltStack, StackStorm


IOS-XR uses pyIOSXR:
https://github.com/napalm-automation/napalm/blob/develop/napalm/iosxr/iosxr.py

Junos uses PyEZ:
https://github.com/napalm-automation/napalm/blob/develop/napalm/junos/junos.py

EOS uses pyeapi:
https://github.com/napalm-automation/napalm/blob/develop/napalm/eos/eos.py


Closing
============

my connection
- SaltStack
- IOS XR
- Junos
- Arista