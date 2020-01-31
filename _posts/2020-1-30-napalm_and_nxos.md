---
layout: post
title: NAPALM and NX-OS
tags: [automation, python, napalm, cisco ]
image: /img/napalm_logo.png
---


Recently, I have been exploring NAPALM in relation to NX-OS. To connect to the device, I used the <b>nxos</b> driver which uses the NXAPI. This writeup contains some of the basic operations. I am using NAPALM version 2.5.0 and the NX-OS device is a Nexus7700 running 8.0(1).


## Enabling the NXAPI

First we need to enable the API on the NX-OS. In addition to that, NAPALM also requires the <b>feature scp-server</b> feature turned on. To this end, we configure the following on the device:


<pre style="font-size:12px">
feature scp-server
feature nxapi
no nxapi http
nxapi https port 65000
</pre>

We disabled HTTP access and we specified a non-standard port for HTTPs. We can verify our configuration with the following command:


<pre style="font-size:12px">
nxos.lab# <b>show feature | egrep "nxapi|scp"</b>
nxapi                1        enabled
scpServer            1        enabled

nxos.lab# <b>show nxapi</b>

NX-API:       Enabled         Sandbox:      Disabled    
HTTP Port:    Disabled        HTTPS Port:   65000    
</pre>


## Retrieving information from the device

To verify that we can connect to the device, let's start out issuing a CLI command:

```python
import napalm

optional_args = {'port': '65000' }
driver = napalm.get_network_driver('nxos')

device = driver(hostname='192.0.2.1', username='admin', password='admin123', optional_args=optional_args )
device.open()
return_dictionary = device.cli(['show ipv6 ospfv3 neighbors', ])
device.close()

s = return_dictionary['show ipv6 ospfv3 neighbors']

print(s)
```

Since we are using a non-standard port, we use the <b>optional_args</b> when we instantiate the device object to specify the port number. Note that by default, HTTPS is used for transport. If for whatever reason you absolutely want to use HTTP, then you can use the following: <b>optional_args = {'transport': 'http' }</b>.

When we run the script, we get the following output:

<pre style="font-size:12px">
 OSPFv3 Process ID 20 VRF default
 Total number of neighbors: 3
 Neighbor ID     Pri State            Up Time  Interface ID    Interface
 10.47.118.253    1 FULL/ -          3y0w     90              Vlan2 
   Neighbor address fe80::8e60:4fff:fee9:1141
 10.47.118.243  128 FULL/ -          2y16w    4               Po3 
   Neighbor address fe80::86c1:c1ff:fee5:fc5
 10.47.118.244  128 FULL/ -          2y16w    4               Po4 
   Neighbor address fe80::86c1:c1ff:feb5:78c7
</pre>

When looking at the base NAPALM NetworkDriver class, I found that it has the `__enter__` and `__exit__` magic methods implemented. Because of this, we can implement objects using the <b>with</b> statement. When you use this approach, there is no need to specifically open of close the connection:

```python
import napalm
from pprint import pprint as pp

optional_args = {'port': '65000' }
driver = napalm.get_network_driver('nxos')

with driver(hostname='192.0.2.1', username='admin', password='admin123', optional_args=optional_args ) as device:
  return_dictionary = device.cli(['show ipv6 ospfv3 neighbors', ])

s = return_dictionary['show ipv6 ospfv3 neighbors']

print(s)
```

There are some basic methods available to us 'out of the box':

```python
import napalm
from pprint import pprint as pp

optional_args = {'port': '65000' }
driver = napalm.get_network_driver('nxos')

with driver(hostname='192.0.2.1', username='admin', password='admin123', optional_args=optional_args ) as device:  
  pp(device.get_facts())
  pp(device.get_bgp_neighbors())
  pp(device.get_lldp_neighbors())
```

The nice thing about those methods is that they return structured data. Not everything is available out of the box though. The OSPF neighbors for instance are not among the 'getters'. One quick and easy way to retrieve structured data from the NX-OS is through the use of the <b>| json</b> option that is available to several CLI commands.


```python
import napalm
from pprint import pprint as pp
import json

optional_args = {'port': '65000' }
driver = napalm.get_network_driver('nxos')

with driver(hostname='192.0.2.1', username='admin', password='admin123', optional_args=optional_args ) as device:
  ospf3_information = device.cli(['show ipv6 ospfv3 neighbors | json', ])

ospf_dictionary = json.loads(ospf3_information['show ipv6 ospfv3 neighbors | json'])

pp(ospf_dictionary)
```

When we run the above example, we get the following output:
<pre style="font-size:12px">
{'TABLE_ctx': {'ROW_ctx': {'TABLE_nbr': {'ROW_nbr': [{'addr': 'fe80::8e60:4fff:fee9:1141',
                                                      'drstate': '-',
                                                      'ifid': '90',
                                                      'intf': 'Vlan2',
                                                      'priority': '1',
                                                      'rid': '10.47.118.253',
                                                      'state': 'FULL',
                                                      'uptime': 'P3Y17DT3H53M19S'},
                                                     {'addr': 'fe80::86c1:c1ff:fee5:fc5',
                                                      'drstate': '-',
                                                      'ifid': '4',
                                                      'intf': 'Po3',
                                                      'priority': '128',
                                                      'rid': '10.47.118.243',
                                                      'state': 'FULL',
                                                      'uptime': 'P2Y4M5DT16H45M22S'},
                                                     {'addr': 'fe80::86c1:c1ff:feb5:78c7',
                                                      'drstate': '-',
                                                      'ifid': '4',
                                                      'intf': 'Po4',
                                                      'priority': '128',
                                                      'rid': '10.47.118.244',
                                                      'state': 'FULL',
                                                      'uptime': 'P2Y4M5DT16H20M31S'}]},
                           'cname': 'default',
                           'nbrcount': '3',
                           'ptag': '20'}}}
</pre>

NX-OS does not have this option implemented for all CLI commands, so your mileage may vary. The <b>show ip interfaces brief | json</b> command is supported for instance, but the <b>show ip bgp summary | json</b> is not. In those cases, it might help to look at textFSM or see if you can get by with some simple string methods.


## Configuring the device

Let's start out configuring the following ACL:

<pre style="font-size:12px">
ipv6 access-list management-v6
  10 remark hq
  11 permit ipv6 2001:db8:1200:8800::/59 any
  20 remark branch
  21 permit ipv6 2001:db8:2200:8800::/59 any
</pre>

```python
import napalm
driver = napalm.get_network_driver('nxos')

with driver(hostname='192.0.2.1', username='admin', password='admin123', optional_args={'port': '65000' } ) as device:
  device.load_merge_candidate(filename='/var/tmp/add_acl.cfg')
  print(device.compare_config())
  device.commit_config()  
```

When we run the script, the ACL is configured and the diff gets printed to screen:

<pre style="font-size:12px">
sh-4.4# python3 nx_os_add_acl.py 
ipv6 access-list management-v6
  10 remark hq
  11 permit ipv6 2001:db8:1200:8800::/59 any
  20 remark branch
  21 permit ipv6 2001:db8:2200:8800::/59 any
</pre>

This diff is informative, but it is unlike the diff you get from Juniper, Arista or IOSXR. When you use the merge operation in NAPALM, this diff is a simple comparison between the configuration that you are loading and the running configuration.

Let's update the ACL with the following:

<pre style="font-size:12px">
sh-4.4# cat /var/tmp/acl_update.cfg   
no ipv6 access-list management-v6
ipv6 access-list management-v6
  10 remark hq
  11 permit ipv6 2001:DB8:1200:8800::/59 any
</pre>


The diff will output the following:

<pre style="font-size:12px">
no ipv6 access-list management-v6
  11 permit ipv6 2001:DB8:1200:8800::/59 any
</pre>

The 'no xxx' is not in the configuration, so that is displayed. In the 11th sequence, I used 'DB' instead of 'db' so that is something that differs. The lines of the ACL that get removed are not detected by the <b>_get_merge_diff()</b> function.

Thing worth noting is that when you use the <b>load_replace_candidate()</b>, the way the diff happens changes. What will happen in that case is it will create a checkpoint file called <b>sot_file</b>. After this, it will perform a <b>show diff rollback-patch file sot_file file <i>candidate_cfg.txt</i></b>, where the candidate_cfg.txt is the configuration that you upload to the device. This should be a valid checkpoint configuration that is extended with the configuration that you want to add. 

The diff will be better (not perfect), but it is a finicky process at best. It will make you appreciate Junos, EOS or IOSXR all the more.



Ps, to kill the warnings you might see when you are connecting over HTTPS, you can add the following snippet to your script:


```python
import urllib3
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
```