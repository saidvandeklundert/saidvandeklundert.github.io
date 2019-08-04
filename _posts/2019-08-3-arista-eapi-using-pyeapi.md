---
layout: post
title: Exploring the Arista eAPI using pyeapi.
image: /img/arista_logo.jpg
---


The Arista EOS command API, or <b>eAPI</b>, has been maturing ever since EOS 4.12. Recently, I started to look into ways to use this API for information retrieval. In this article, I am sharing what I learned exploring the eAPI using a library called <b>pyeapi</b>.

<br>

Enabling the API
================


To enable the API on the device and allow access from a vrf called `lab`, I used the following configuration:

<pre>
management api http-commands
   no shutdown
   !
   vrf labmgmt
      no shutdown
</pre>

You can verify that the API is enabled using `show management api http-commands`. The resulting output is pretty self-explanatory:

<pre>
lr.lon01#show management api  http-commands
Enabled:            Yes
< output omitted >
VRFs:               lab
</pre>

<br>

Connecting to the device
========================


After installing `pyeapi` using `pip install pyeapi`, we can use the following to send a command to the device:

```python
import pyeapi
import pprint

eapi_param = pyeapi.client.connect(
    transport='https',
    host='lr.lon01',
    username='salt',
    password='salt123',
    port=443,
)
eapi = pyeapi.client.Node(eapi_param)

version_info = eapi.run_commands(['show version',])
pprint.pprint(version_info)
```

Here, we pass the `eapi_param` to `pyeapi.client.Node()`. As a result, we can use the `eapi` object to communicate with the device. Then we call the `run_commands` method and use it to send the `show version` command to a device. The output is stored in `version_info` which is then printed to screen. 

Running the script gives us the following output:

```python
[{u'architecture': u'i386',
  u'bootupTimestamp': 1527495886.66,
  u'hardwareRevision': u'11.00',
  u'internalBuildId': u'54acf3af-762b-4cc0-83b7-3b2060a5c8c1',
  u'internalVersion': u'4.18.0F-4224134.4180F',
  u'isIntlVersion': False,
  u'memFree': 5346020,
  u'memTotal': 8008976,
  u'modelName': u'DCS-7260CX-64-F',
  u'serialNumber': u'JPE18280004',
  u'systemMacAddress': u'44:4c:a8:42:5e:45',
  u'version': u'4.18.0F'}]
```

Structured data, great! No more need to create `textfsm` templates.

When I started out, the thing I was most interested in was verifying the mlag status in a script. So the first thing I did after `show version` was add `show mlag`:

```python
mlag_info = eapi.run_commands(['show mlag',])
pprint.pprint(mlag_info)
```

The resulting output from this addition to the script was the following:

```python
[{u'configSanity': u'consistent',
  u'domainId': u'lr01',
  u'localInterface': u'Vlan4000',
  u'localIntfStatus': u'up',
  u'mlagPorts': {u'Active-full': 7,
                 u'Active-partial': 1,
                 u'Configured': 0,
                 u'Disabled': 0,
                 u'Inactive': 3},
  u'negStatus': u'connected',
  u'peerAddress': u'192.254.1.2',
  u'peerLink': u'Port-Channel10',
  u'peerLinkStatus': u'up',
  u'portsErrdisabled': False,
  u'reloadDelay': 300,
  u'reloadDelayNonMlag': 300,
  u'state': u'active',
  u'systemId': u'47:3c:b8:44:11:29'}]
```

Familiarity with the CLI will allow you to quickly add whatever you need. Another thing worth noting is that you can use `| json` on the CLI to figure out the reponse from the eAPI in advance:


<pre>
lr.lon01#show mlag | json 
{
    "localInterface": "Vlan4000",
    "systemId": "47:3c:b8:44:11:29",
    "domainId": "lr01",
    "peerLink": "Port-Channel10",
    "localIntfStatus": "up",
    "peerLinkStatus": "up",
    "peerAddress": "192.254.1.2",
    "configSanity": "consistent",
    "portsErrdisabled": false,
    "state": "active",
    "reloadDelay": 300,
    "reloadDelayNonMlag": 300,
    "negStatus": "connected",
    "mlagPorts": {
        "Disabled": 0,
        "Active-partial": 1,
        "Inactive": 3,
        "Configured": 0,
        "Active-full": 7
    }
}
lr.lon01#
</pre>

Good stuff! This is a nice way to see what command you should put in your script and what return value you will be dealing with. 

<br>

Using the API module
====================


So far, we have been sending CLI commands to the device. But Arista also offers <b>API modules</b> to retrieve and manipulate configuration. Let’s look at an example script where we inspect the MLAG configuration of a device: 

```python
import pyeapi
import pprint

eapi_param = pyeapi.client.connect(
    transport='https',
    host='lr.lon01',
    username='salt',
    password='salt123',
    port=443,
)
eapi = pyeapi.client.Node(eapi_param)

mlag = eapi.api('mlag')
mlag_d = mlag.get()
pprint.pprint(mlag_d)
```

Previously, we used the `run_commands` method that the `'pyeapi.client.Node'` has to offer. This time, we call the `api` method for `mlag` using `mlag = eapi.api('mlag')`.

The `mlag` API gives us several methods to work with ( https://pyeapi.readthedocs.io/en/latest/api_modules/mlag.html ). We simply use `get` to check the configuration of the mlag using `mlag_d = mlag.get()`. 

When we run the script, we get the following:

```python
{'config': {'domain_id': None,
            'local_interface': 'Vlan4000',
            'peer_address': '192.254.1.2',
            'peer_link': 'Port-Channel10',
            'shutdown': False},
 'interfaces': {'Port-Channel11': {'mlag_id': '11'},
                'Port-Channel12': {'mlag_id': '12'},
                'Port-Channel13': {'mlag_id': '13'},
                'Port-Channel14': {'mlag_id': '14'},
                'Port-Channel15': {'mlag_id': '15'},
                'Port-Channel16': {'mlag_id': '16'},
                'Port-Channel17': {'mlag_id': '17'},
                'Port-Channel18': {'mlag_id': '18'},
                'Port-Channel5': {'mlag_id': '5'},
                'Port-Channel6': {'mlag_id': '6'},
                'Port-Channel7': {'mlag_id': '7'}}}
>>> 
```

This gives us the MLAG configuration in dictionary format. 

There are plenty of other APIs available to us in addition to `mlag` (https://pyeapi.readthedocs.io/en/latest/api_modules/_list_of_modules.html). 

Some insight into how easy it is to use the `get` method on the other APIs:

```python
ospf = eapi.api('ospf')
ospf_d = ospf.get()

bgp = eapi.api('bgp')
bgp_d = bgp.get()

acl = eapi.api('acl')
acl_d = acl.getall()
```

<br>

Closing thoughts
================


The eAPI certain looks to be of great value for anything related to operations. I like how easy it is to translate your CLI routines into a script and how you can use the CLI to discover how output will be presented in JSON. Working out how to automate tasks currently done from the CLI is pretty easy.

Additionally, using the API to scan through configuration is easy and it beats string methods and regex I have been using up untill now. I'll save working with the other methods available for another time.

Note: according to the documentation, `pyeapi` support for Python 3 is in the works. For now though, it is Python 2.7 only.



==


Let’s check out `eapi` in more detail:

```python
>>> type(eapi)
<class 'pyeapi.client.Node'>
>>> eapi
Node(connection=EapiConnection(transport=https://lr.lon01:443//command-api))
```
```
>>> dir(eapi)
['__class__', '__delattr__', '__dict__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_connection', '_enablepwd', '_get_version_properties', '_model', '_running_config', '_startup_config', '_version', '_version_number', 'api', 'autorefresh', 'config', 'connection', 'enable', 'enable_authentication', 'get_config', 'model', 'refresh', 'run_commands', 'running_config', 'section', 'settings', 'startup_config', 'version', 'version_number']
```



