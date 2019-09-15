---
layout: post
title: NAPALM
image: /img/napalm_logo.png.png
---

Up until this point in my career, most of my scripting efforts involved <b>PyEZ</b>, <b>Paramiko</b> and <b>Netmiko</b>. But after having to work with multiple vendors inside <b>SaltStack</b>, I started looking into <b>NAPALM</b>. This post is a short summary of what I have learned so far about NAPALM in general.  

<p align="center">
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_logo.png">
</p>

<br>

NAPALM in a nutshell
====================

Every networking vendor has its own operating system and all those operating systems have their own APIs. In case you are running a multivendor network, one of the challenges you will run into when you want to start automating, is figuring out how all these different APIs work.

Wouldn’t it be nice if there was some sort of library you could import and have that library deal with all those differences? Like a wrapper around all the vendor-specific things.

Enter <b>NAPALM</b>.

Though it does not abstract everything away, it is quite convenient for a lot of basic operations and it will enable you to get somewhere quickly.


<br>

What devices does NAPALM support?
=================================


At the time of writing, the [docs](https://napalm.readthedocs.io/en/latest/) indicate that NAPALM offers support for the following devices:
- Arista EOS
- Cisco IOS
- Cisco IOS-XR
- Cisco NX-OS
- Juniper JunOS

NAPALM uses a set of <b>core drivers</b> to be able to deal with all these vendors. These core drivers are the backend libraries that NAPALM uses to communicate with different operating systems. Currently, the core libraries are the following:

  
  |                     | EOS       | Junos         | IOS-XR     | NX-OS     | NX-OS SSH  | IOS
  | ------------------- | --------- | ------------- | ---------- | --------- | ---------- | ---- 
  | **Driver Name**     | eos       | junos         | iosxr      | nxos      | nxos_ssh   | ios
  | **Backend library** | [pyeapi](https://github.com/arista-eosplus/pyeapi)  | [junos-eznc](https://github.com/Juniper/py-junos-eznc)  | [pyIOSXR](https://github.com/fooelisa/pyiosxr)  | [pynxos](https://github.com/networktocode/pynxos)  | [netmiko](https://github.com/ktbyers/netmiko)  | [netmiko](https://github.com/ktbyers/netmiko)
  

Notice that in NAPALM, Netmiko is used to get NAPALM to talk to NX-OS and IOS over SSH. In the case of IOS, this is as good as it get's due to the fact that Cisco never made an API for that OS.

Another thing worth mentioning is that you can also choose to use these backend libraries directly in your scripts. This can be quite usefull to know when you run into situations where NAPALM does not have a certain function available for you. 

Finally, in addition to the core drivers, which are supported and maintained by the people working on NAPALM, there are also various <b>community drivers</b>. Community drivers are maintained under their own repository and can be used by NAPALM. I found a list of community drivers [here](https://github.com/napalm-automation-community).

<br>

Using NAPALM in a Python script 
===============================

NAPALM offers you a variety of methods that gather information from devices and another set of methods that can help you with your configuration efforts. There a few really nice things about these methods. First of all, you can use them against all the supported vendors. You only have to make sure that you select the proper driver when you use NAPALM to connect to a device. In addition to that, the information that is returned by NAPALM is structured in the same way reagardless of the vendor you request the information from. And lastly, again, you are not bothered by the vendor-specifics of these devices.

Let’s explore the use of several methods NAPALM has to offer when working with Juniper and Cisco (IOS XR).

<br>

## Using NAPALM for information retrieval

In this example, we will have NAPALM talk to a Juniper and a Cisco device and display information that describes the device. In addition to that, we will also have it display information about the BGP neighbors.

Doing this using NAPALM is very easy. All we need to do the following:
- import napalm
- select the proper driver
- use `get_facts` and `get_bgp_neighbors`

<p align="center" >
  <img src="https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/napalm/img/napalm_information_example.png">
</p>

The Python required to perform the above when connecting to a Juniper device could be something like the following `junos-example.py`:

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

Because we are connecting to a Juniper device, we selected the `junos` driver. This will make NAPALM use the `junos-eznc` backend library to communicate with the Juniper XML API. When we run the script using `python junos-example.py`, we get the following:

```python           
{u'fqdn': u'vmx06',
 u'hostname': u'vmx06',
 u'interface_list': ['ge-0/0/1', '.local.', 'lo0', 'lsi'],
 u'model': u'VMX',
 u'os_version': u'19.1R1.6',
 u'serial_number': u'VM5CB1E69517',
 u'uptime': 12859273,
 u'vendor': u'Juniper'}
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
                                       u'uptime': 1393131},
                        u'10.0.0.15': {u'address_family': {u'ipv4': {u'accepted_prefixes': -1,
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
                                       u'uptime': 1297505}},
             u'router_id': u'10.0.0.6'}}         
```

Without having to write any complex code, we get some information about the device as well as some information about the BGP peers that the device is configured with.

So what happens in the script? After importing the libraries, we specified what driver we wanted to use and instantiated a device object:

```python
driver = napalm.get_network_driver('junos')
device = driver(hostname='169.50.169.100', username='salt', password='salt123')
```

After this, we used several NAPALM methods:

```python
device.open()
pp(device.get_facts())
pp(device.get_bgp_neighbors())
device.close()
```

The `device.open()` and `device.close()` were used to intiate and close a connection to the device. The `device.get_facts()` and `device.get_bgp_neighbors()` were used to retrieve information from the device.

If we were to check the log on the device during the time where we gathered the BGP neighbor information, we can see the following:

```
Sep  9 13:20:50  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-instance-information'
Sep  9 13:20:50  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-bgp-neighbor-information instance=master'
Sep  9 13:20:50  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-bgp-summary-information instance=master'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-bgp-neighbor-information instance=cust-1'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-bgp-summary-information instance=cust-1'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-bgp-neighbor-information instance=mgmt_junos'
Sep  9 13:20:51  vmx01 mgd[10297]: UI_NETCONF_CMD: User 'salt' used NETCONF client to run command 'get-bgp-summary-information instance=mgmt_junos'
```
<br>

These were all the RPCs our NAPALM script called. NAPALM completely hides the fact that this happened and the nice thing is that you do not need to know any specifics about the Juniper API. NAPALM issued these RPCs, parsed the return values and created a nice dictionary for us to work with.

Let's change the example code to something that works for IOS XR and create a new `iosxr-example.py` script:

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

The only thing we needed to do was change the driver from `junos` to `iosxr` and we are good to go! When we run our new `iosxr-example.py`, we get the following:

```python
{u'fqdn': u'iosxr_1',
 u'hostname': u'iosxr_1',
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
 u'uptime': 269989,
 u'vendor': u'Cisco'}
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
                                       u'uptime': 269826},
                        u'10.0.0.15': {u'address_family': {u'ipv4': {u'accepted_prefixes': 6,
                                                                     u'received_prefixes': 6,
                                                                     u'sent_prefixes': 2}},
                                       u'description': u'',
                                       u'is_enabled': False,
                                       u'is_up': True,
                                       u'local_as': 1,
                                       u'remote_as': 1,
                                       u'remote_id': u'10.0.0.15',
                                       u'uptime': 269826}},
             u'router_id': u'10.0.1.1'}}
```



In the previous examples, I used `get_facts` and `get_bgp_neighbors`. But these are not the only `getters` that NAPALM provides us with. There are many more [NAPALM getters](https://napalm.readthedocs.io/en/latest/support/index.html#getters-support-matrix) for you to check out. A nice way to do this is by trying them out. A quick way to do this is by running them from the interpreter.


```python
>>> import napalm
>>> driver = napalm.get_network_driver('iosxr')
>>> device = driver(hostname='169.50.169.101', username='salt', password='salt123')
>>> 
>>> dir(device)
['__class__', '__del__', '__delattr__', '__dict__', '__doc__', '__enter__', '__exit__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_canonical_int', '_netmiko_close', '_netmiko_open', 'cli', 'close', 'commit_config', 'compare_config', 'compliance_report', 'connection_tests', 'device', 'discard_config', 'get_arp_table', 'get_bgp_config', 'get_bgp_neighbors', 'get_bgp_neighbors_detail', 'get_config', 'get_environment', 'get_facts', 'get_firewall_policies', 'get_interfaces', 'get_interfaces_counters', 'get_interfaces_ip', 'get_ipv6_neighbors_table', 'get_lldp_neighbors', 'get_lldp_neighbors_detail', 'get_mac_address_table', 'get_network_instances', 'get_ntp_peers', 'get_ntp_servers', 'get_ntp_stats', 'get_optics', 'get_probes_config', 'get_probes_results', 'get_route_to', 'get_snmp_information', 'get_users', 'hostname', 'is_alive', 'load_merge_candidate', 'load_replace_candidate', 'load_template', 'lock_on_connect', 'netmiko_optional_args', 'open', 'password', 'pending_changes', 'ping', 'platform', 'port', 'post_connection_tests', 'pre_connection_tests', 'replace', 'rollback', 'timeout', 'traceroute', 'username']
>>> 
>>> device.open()
>>> device.get_interfaces_ip()
{u'GigabitEthernet0/0/0/1.10': {u'ipv4': {u'172.0.2.1': {u'prefix_length': 30}, u'10.0.2.1': {u'prefix_length': 31}}}, u'GigabitEthernet0/0/0/0.1156': {u'ipv4': {u'169.50.169.101': {u'prefix_length': 28}}}, u'GigabitEthernet0/0/0/3.2002': {u'ipv4': {u'10.0.0.9': {u'prefix_length': 30}}, u'ipv6': {u'2001:db8:1::1': {u'prefix_length': 127}}}, u'Loopback0': {u'ipv4': {u'10.0.1.1': {u'prefix_length': 32}}}, u'GigabitEthernet0/0/0/2.12': {u'ipv4': {u'10.0.2.5': {u'prefix_length': 31}}}}
>>> device.get_arp_table()
<output omitted>
>>> device.get_bgp_config()
<output omitted>
>>> device.get_bgp_neighbors()
<output omitted>
>>> device.get_bgp_neighbors_detail()
<output omitted>
>>> device.get_config()
<output omitted>
>>> device.get_interfaces_ip()
<output omitted>
>>> device.get_lldp_neighbors()
<output omitted>
```

With `dir(device)`, we listed all the methods we have available. After having opened the connection to the device using `device.open()`, we can see that examining the output it is just a matter of issuing the 'getter' methods one by one.

<br>

## Using NAPALM to configure devices

We can also use NAPALM to [configure devices](https://napalm.readthedocs.io/en/latest/support/index.html#configuration-support-matrix). 

We can choose to manipulate the configuration using either of:
- <b>replace</b>: puts in a completely configuration and removes the all of the existing configuration 
- <b>merge</b>: add configuration statements to the current configuration

Using the previous methods to manipulate the configuration does not change the running, or active, configuration on the device. It alters a candidate (in Junos speak) configuration but it does not commit this. Once we have altered this candidate configuration, we can use the following NAPALM methods:
- <b>compare</b>: see how the configuration would look _if_ you were to apply configuration statements. 
- <b>rollback</b>: change the device configuration to a previous version
- <b>commit</b>: apply any staged configuration changes to the device
- <b>discard</b>: abort configuration efforts and 'discard' the staged configuration

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
