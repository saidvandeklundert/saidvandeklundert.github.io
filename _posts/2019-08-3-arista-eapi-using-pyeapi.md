---
layout: post
title: Exploring the Arista eAPI using pyeapi.
image: /img/arista_logo.jpg
---


The Arista EOS command API, or <b>eAPI</b>, has been maturing ever since eos 4.12. Recently, I started to look into ways to use this API for information retrieval. In this article, I am sharing what I learned exploring the eAPI using a library called <b>pyeapi</b>.

<br>

Enabling the API
================


To enable the API on the device, we can apply the following configuration:

<pre>
configure terminal
management api http-commands
no shutdown
</pre>

In case you are accessing the device through an interface that is placed inside a vrf, you need to add configuration to allow API access from that vrf as well. In the following example, we enable the API for the `lab` vrf:

<pre>
   vrf lab
      no shutdown
</pre>

You can verify that the API is working using `show management api http-commands`. The resulting output is pretty self-explanatory:

<pre>
lr.lon01#show management api  http-commands
Enabled:            Yes
< output omitted >
VRFs:               lab
</pre>

After installing `pyeapi` using `pip install pyeapi`, we are now good to go. 

Note: according to the documentation, support for Python 3 is in the works. For now though, it is Python 2 only.

<br>

Connecting to the device
========================


A lot of the Arista examples in the documentation seem to steer you towards creating and using a configuration file. I do not really like that and luckily, we do not have to do this. 

The following script will allow us to send a command to the device:

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

Here, we pass the `eapi_param` to `pyeapi.client.Node()`. As a result, the `eapi` object represents a single device that we can exchange eAPI messages with.
 
After having done this, we call the `run_commands` method, which enables us to send a list of commands to the device. The output is stored in `version_info` and finally, we print that to screen.

So let’s run the example script and examine the output:

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

We get a dictionary in a list, great! 

We can use this example to execute other commands as well. Additionally, while we are on the CLI we can figure out in advance what the response will be using `| json`:


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

Let’s add the following to the script:

```python
mlag_info = eapi.run_commands(['show mlag',])
pprint.pprint(mlag_info)
```

The resulting output from this addition would be as follows:

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

Good stuff. Working out how to automate some tasks currently done from the CLI is pretty easy.

<br>

Using the API module
====================


So far, we have been sending CLI commands to the device. But Arista also offers API modules to retrieve and manipulate configuration. Let’s look at an example script where we inspect the MLAG configuration of a device: 

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

Let’s run the script in interactive mode:

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

We get to see the MLAG configuration in dictionary format. So what exactly happened?

Let’s check out `eapi` in more detail:

```python
>>> type(eapi)
<class 'pyeapi.client.Node'>
>>> eapi
Node(connection=EapiConnection(transport=https://lr.lon01:443//command-api))
>>> dir(eapi)
['__class__', '__delattr__', '__dict__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_connection', '_enablepwd', '_get_version_properties', '_model', '_running_config', '_startup_config', '_version', '_version_number', 'api', 'autorefresh', 'config', 'connection', 'enable', 'enable_authentication', 'get_config', 'model', 'refresh', 'run_commands', 'running_config', 'section', 'settings', 'startup_config', 'version', 'version_number']
```

So we see `eapi` is a class with several methods. Previously, we used `run_commands`. This time though, we will use `api`. The APIs available to us is found here: https://pyeapi.readthedocs.io/en/latest/api_modules/_list_of_modules.html.

In our script,  we only checked the `mlag` api using `mlag = eapi.api('mlag')`:

```python
>>> type(mlag)
<class 'pyeapi.api.mlag.Mlag'>
>>> mlag
<pyeapi.api.mlag.Mlag object at 0x7f81eba0a350>
>>> dir(mlag)
['__abstractmethods__', '__call__', '__class__', '__delattr__', '__dict__', '__doc__', '__format__', '__getattribute__', '__hash__', '__init__', '__metaclass__', '__module__', '__new__', '__reduce__', '__reduce_ex__', '__repr__', '__setattr__', '__sizeof__', '__str__', '__subclasshook__', '__weakref__', '_abc_cache', '_abc_negative_cache', '_abc_negative_cache_version', '_abc_registry', '_configure_mlag', '_parse_config', '_parse_domain_id', '_parse_interfaces', '_parse_local_interface', '_parse_peer_address', '_parse_peer_link', '_parse_shutdown', 'command_builder', 'config', 'configure', 'configure_interface', 'error', 'get', 'get_block', 'node', 'set_domain_id', 'set_local_interface', 'set_mlag_id', 'set_peer_address', 'set_peer_link', 'set_shutdown']
```

We have several things to work with here. The description of all these methods is found here: https://pyeapi.readthedocs.io/en/latest/api_modules/mlag.html

In our example, we are using `get` to check the configuration of the mlag. We did that using `mlag_d = mlag.get()` and this gave us:

```python
>>> type(mlag_d)
<type 'dict'>
>>> mlag_d
{'interfaces': {'Port-Channel5': {'mlag_id': '5'}, 'Port-Channel6': {'mlag_id': '6'}, 'Port-Channel7': {'mlag_id': '7'}, 'Port-Channel12': {'mlag_id': '12'}, 'Port-Channel13': {'mlag_id': '13'}, 'Port-Channel11': {'mlag_id': '11'}, 'Port-Channel16': {'mlag_id': '16'}, 'Port-Channel17': {'mlag_id': '17'}, 'Port-Channel14': {'mlag_id': '14'}, 'Port-Channel15': {'mlag_id': '15'}, 'Port-Channel18': {'mlag_id': '18'}}, 'config': {'shutdown': False, 'peer_address': '192.254.1.2', 'local_interface': 'Vlan4000', 'domain_id': None, 'peer_link': 'Port-Channel10'}}
```

Some insight into how easy it is to extend the script to other parts of the configuration:

```python
ospf = eapi.api('ospf')
ospf_d = ospf.get()

bgp = eapi.api('bgp')
bgp_d = bgp.get()

acl = eapi.api('acl')
acl_d = acl.getall()
```

Using the API to scan through configuration is pretty nice, and it beats string methods and regex! I'll save working with the other methods available for another time.

<br>

Closing thoughts
================


The eAPI certain looks to be of great value for anything related to operations. I like how easy it is to translate your CLI routines into a script and how you can use the CLI to discover how output will be presented in JSON. 



