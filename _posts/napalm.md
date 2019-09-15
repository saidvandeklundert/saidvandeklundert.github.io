---
layout: post
title: NAPALM
image: /img/napalm_logo.png.png
---

Up until this point, most of my scripting efforts involved <b>PyEZ</b>, <b>Paramiko</b> and <b>Netmiko</b>. But after having to work with multiple vendors inside <b>SaltStack</b>, I started looking into <b>NAPALM</b>. This post is a short summary of what I have learned so far about NAPALM in general.  

<p align="center">
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_logo.png">
</p>
<br>

NAPALM in a nutshell
====================

Different networking vendors each have their own OS that comes with its own API. In case you have a multivendor network, one of the challenges you will run into when you want to start automating, is figuring out how all these different APIs work.

Wouldn’t it be nice if there was some sort of library you could import and be done with it? Like a wrapper around all the vendor-specific things.

Enter <b>NAPALM</b>.

Though it does not abstract everything away, it is quite convenient for a lot of basic operations and it will enable you to really get somewhere quickly.


<br>

What devices does NAPALM support?
=================================


At the time of writing, the [docs](https://napalm.readthedocs.io/en/latest/) indicate that NAPALM offers support for the following devices:
- Arista EOS
- Cisco IOS
- Cisco IOS-XR
- Cisco NX-OS
- Juniper JunOS

These devices are managed through the 'core' drivers that are supported and maintained by the people working on NAPALM. These drivers are libraries that have been written to deal with the differences that exist between the vendor APIs. The following table displays the current core libraries:

  
  |                     | EOS       | Junos         | IOS-XR     | NX-OS     | NX-OS SSH  | IOS
  | ------------------- | --------- | ------------- | ---------- | --------- | ---------- | ---- 
  | **Driver Name**     | eos       | junos         | iosxr      | nxos      | nxos_ssh   | ios
  | **Backend library** | [pyeapi](https://github.com/arista-eosplus/pyeapi)  | [junos-eznc](https://github.com/Juniper/py-junos-eznc)  | [pyIOSXR](https://github.com/fooelisa/pyiosxr)  | [pynxos](https://github.com/networktocode/pynxos)  | [netmiko](https://github.com/ktbyers/netmiko)  | [netmiko](https://github.com/ktbyers/netmiko)
  

You could even choose to use these backend libraries in your scripts. Two backend libraries were actually pretty familiar to me already. I worked with `junos-eznc`, a.k.a. PyEZ, when managing Juniper devices. And `netmiko` was familiar because of some scripting I did against devices from several different vendors. The `netmiko` library is a ```Multi-vendor library to simplify Paramiko SSH connections to network devices```. In NAPALM though, it is used to get NAPALM to talk to NX-OS (over SSH) and IOS.

In addition to the core drivers, there are also various 'community drivers'. Community drivers are maintained under their own repository and can be used by NAPALM. I found a list of community drivers [here](https://github.com/napalm-automation-community).

<br>

NAPALM in a Python script: 
=========================

The NAPALM drivers allow you to gather information from different vendors and it offers a variety of functions to help you with your configuration efforts. Let's explore how we can use NAPALM for both these topics.

## Using NAPALM for information retrieval

NAPALM provides you with several basic functions that you can use to interact with different vendors. Let's look at an example where we retrieve information from a Juniper and a Cisco device. In the example, we will have NAPALM do the following:
- display information that describes the device
- display information about the BGP neighbors

Doing this using NAPALM is very easy. All we need to do the following:
- import napalm
- select the proper driver
- use `get_facts` and `get_bgp_neighbors`

<p align="center" >
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_information_example.png">
</p>

The Python required to perform the above when connecting to a Juniper device could be something like the following:

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

Because we are connecting to a Juniper device, we selected the `junos` driver. This will make NAPALM use the `junos-eznc` backend library to communicate with the Juniper XML API. When we run this example in the interpreter, we get the following:

```python
bash-4.4$ python
>>> import napalm                  
>>> from pprint import pprint as pp
>>> driver = napalm.get_network_driver('junos')
>>> device = driver(hostname='169.50.169.171', username='salt', password='salt123')
>>> device.open()
>>> pp(device.get_facts())
{u'fqdn': u'vmx6',
 u'hostname': u'vmx6',
 u'interface_list': ['ge-0/0/1', '.local.', 'lo0', 'lsi'],
 u'model': u'VMX',
 u'os_version': u'19.1R1.6',
 u'serial_number': u'VM5CB1EEE555',
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

NAPALM manages to completely hide this fact and the beauty here is that you do not need to know all the specifics with regards to the Juniper API. NAPALM issued these RPCs, parsed the return values and created a nice dictionary for us to work with.

Let's change the example code to something that works for IOS XR:

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

When we run this in the interpreter, we get the following:

```python
bash-4.4$ python
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
 u'serial_number': u'9E478CCC333',
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

The only thing we needed to do was change the driver from `junos` to `iosxr` and we were good to go!

In the example, I used `get_facts` and `get_bgp_neighbors`. But these are not the only `getters` that NAPALM provides us with. There are many more [NAPALM getters](https://napalm.readthedocs.io/en/latest/support/index.html#getters-support-matrix) for you to check out. 

## Using NAPALM to configure devices

We can also use NAPALM to [configure devices](https://napalm.readthedocs.io/en/latest/support/index.html#configuration-support-matrix). 

When we do this, some of the options available to us are the following:
- replace: puts in a completely configuration and removes the all of the existing configuration 
- merge: add configuration statements to the current configuration
- compare: see how the configuration would look _if_ you were to apply configuration statements. 
- rollback: change the device configuration to a previous version
- commit: apply any staged configuration changes to the device
- discard: abort configuration efforts and 'discard' the staged configuration

As noted in the NAPALM documentation, not all options are available on every device and there are some caveats to using this. 

However, it comes in very handy on devices that do support these capabilities. I have successfully used several of these functions on Juniper, Arista and Cisco IOS XR. All of the aforementioned vendors have a slightly different approach and they all have their own terminology. The backend libraries hide all these things and allow us to use the same code to talk to these different vendors without having to write or maintain any Python of our own.


Next is a short example on how we could use NAPALM when configuring a Cisco IOS XR. In the example, we will use the following configuration file:

```bash
bash-4.4$ cat /var/tmp/iosxr.cfg
interface loopback 0 description description TESTING-1
interface GigabitEthernet0/0/0/1.10 description TESTING-2
interface GigabitEthernet0/0/0/2.12 description TESTING-3
```

In order to load this configuration file, we can do the following:

```python
>>> import napalm
>>> driver = napalm.get_network_driver('iosxr')
>>> device = driver(hostname='169.50.169.170', username='salt', password='salt123')
>>> device.open()
>>> 
>>> device.load_merge_candidate(filename='/var/tmp/iosxr.cfg')
>>> 
>>> print(device.compare_config())
--- 
+++ 
@@ -44,6 +44,7 @@
  10 permit ipv4 host 10.0.1.1 any
 !
 interface Loopback0
+ description description TESTING-1
  ipv4 address 10.0.1.1 255.255.255.255
 !
 interface MgmtEth0/RP0/CPU0/0
@@ -61,6 +62,7 @@
  description common
 !
 interface GigabitEthernet0/0/0/1.10
+ description TESTING-2
  ipv4 address 10.0.2.1 255.255.255.254
  encapsulation dot1q 10
 !
@@ -68,6 +70,7 @@
  description common
 !
 interface GigabitEthernet0/0/0/2.12
+ description TESTING-3
  ipv4 address 10.0.2.5 255.255.255.254
  encapsulation dot1q 12
 !
>>> device.commit_config()
```

If, after loading the configuration, the `compare_config()` would have shown us something that indicated we should back off, we could have used `device.discard_config()` to discard our configuration efforts. 

Same as when we retrieved information from the device, NAPALM takes care of the device specifics here. We do not need to know that IOS XR has a target configuration and that Juniper has a candidate configuration, etc. We can call the same function and use it for the different vendors that NAPALM supports.


<br>

NAPALM outside of a Python script
=================================


As it says on the [NAPALM website](https://napalm-automation.net/), `Napalm plays nice with others!`. Efforts have been made to enable the use of NAPALM outside of a Python script. You can use NAPALM for Ansible, Stackstorm and SaltStack. 

In the case of SaltStack, the automation framework I am most familiar with, this was done through the creation of a proxy minion and several modules that exposes most of the methods available in NAPALM. This ensures you will have an easy time gathering data from supported devices, configuring them and all that even comes included with some grains.

<br>

Closing thoughts
================

What NAPALM has achieved so far is very impressive. A lot of work has been involved in the creation of the backend libraries and getting NAPALM to play nice with several frameworks. 

They have had quite the impact on the networking community and I can see why this is. It is very convenient to work with and you can get things done quickly. 

There is also some risk involved though. Imagine having to go to release X on a product because of some bug  (which is not uncommon). Supose that on this new version, one of the NAPALMs functions no longer works because changes in the API on the vendor side breaks a certain way NAPALM does something. In such cases, at least for a little while, you will be on your own again. 

In addition to this, it is also good to realize that not everything is available through NAPALM. For these reasons, it is still a good idea to educate persons on the team on the device specific APIs and backend libraries that NAPALM, and you, relies on.