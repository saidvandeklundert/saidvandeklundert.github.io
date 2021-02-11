---
layout: post
title: Juniper automation examples for state 
tags: [ python, automation, juniper, pyez, ]
image: /img/juniper_logo.jpg
---


Some commonly used snippets that will help you retrieve information from Juniper devices.

I retrieved the data I will be working with using the following code:

```
#!/usr/bin/python
from lxml import etree
from jnpr.junos import Device
import getpass
import sys

username = sys.argv[1]
host = sys.argv[2]
password = getpass.getpass()

with Device(host=host, user=username, password=password, normalize=True) as dev:                                  
    rpc = dev.rpc.get_interface_information(extensive=True)
```

The `rpc` that is referred to in the rest of the examples is the device response to `get_interface_information(extensive=True)`. This is the RPC to retrieve the information that can be displayed with the `show interfaces extensive` command:

```
admin@ar.lab> show interfaces extensive | display xml rpc 
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/16.1R3/junos">
    <rpc>
        <get-interface-information>
                <extensive/>
        </get-interface-information>
    </rpc>
    <cli>
        <banner></banner>
    </cli>
</rpc-reply>
```

I stored the example script as `example.py` and ran it with `python -i`. This will drop you in the interpretor and allows you to paste in the following snippets and examples.

## XPATH examples:

XPATH getting all the interface names:
```
rpc.xpath('//physical-interface/name/text()')
```

XPATH getting all the logical interface names:
```
rpc.xpath('//physical-interface/logical-interface/name/text()')
```

XPATH getting all the interface names of the physicall as well as the logical interfaces:
```
rpc.xpath('//physical-interface/name/text()|//physical-interface/logical-interface/name/text()')
```

XPATH getting all the physical and logical interface XML data:
```
rpc.xpath('//physical-interface|//physical-interface/logical-interface')
```

Using XPATH, we slice the XML into a list of items. To see the content of the items, that we could do another XPATH expression on, we can use the following:
```
for interface_xml in rpc.xpath('//physical-interface|//physical-interface/logical-interface'):    
    print(etree.tostring(interface_xml))
```

That works, but it does not look very pretty. A (somewhat involved) way to make it look pretty:
```
import xml.dom.minidom
for interface_xml in rpc.xpath('//physical-interface|//physical-interface/logical-interface'):
    xml_minidom = xml.dom.minidom.parseString(etree.tostring(interface_xml))
    interface_xml_pretty = xml_minidom.toprettyxml()
    print(interface_xml_pretty)
```

Even easier then using the above would be to just check the XML on the device.

XPATH getting the physical interface names in case they include 'et':
```
rpc.xpath('physical-interface[contains(name, "et-")]/name/text()')
```

XPATH getting the logical interface names in case they include 'irb':
```
rpc.xpath('physical-interface/logical-interface[contains(name, "irb")]/name/text()') 
```

Using regular expresions in lxml:
```
ns = {"re": "http://exslt.org/regular-expressions"}
# RE to get 100G:
rpc.xpath('physical-interface[re:match(name, "et")]/name/text()', namespaces=ns) 
# RE to get 100G on linecard position 1 and 2:
rpc.xpath('physical-interface[re:match(name, "et-[1,2]")]/name/text()', namespaces=ns)

# RE to get 100G on linecard position 1 and 2 while ignoring upper and lower case:
rpc.xpath('physical-interface[re:match(name, "ET-[1,2]", "i")]/name/text()', namespaces=ns)
```

XPATH to get all interfaces, physical as well as logical:
```
all_interfaces = rpc.xpath('//physical-interface|//physical-interface/logical-interface')
```

Now we can iterate `all_interfaces` to get the name of the interfaces using the `find` method:
```
for interface in all_interfaces:
    # extract the name using find:
    interface.find('./name').text if interface.find('./name') is not None else None
```

The `if interface.find('./name') is not None else None` was added to avoid running into exceptions.

Alternatively, we use an XPATH to accomplish the same thing again:
```
for interface in all_interfaces:
    # extract the name using XPATH:
    interface.xpath('./name/text()') 
```

We can also inspect the text of all the child nodes, just to see what is on offer:
```
for interface in all_interfaces:
    interface.xpath('child::*/text()')
    interface.xpath('child::*/text()')
```

Sometimes, the atributes offer more interested data. In order to access attributes, we can use the following:
```
for interface in all_interfaces:
    interface.find('./interface-flapped').attrib['seconds'] if interface.find('./interface-flapped') is not None else None
```

Safely accessing attributes of can also be done using the Python built-in `getattr`:
```
all_interfaces = rpc.xpath('//physical-interface|//physical-interface/logical-interface')
for interface in all_interfaces:
    getattr(interface.find('./name'), 'text' , '')
    getattr(interface.find('./description'), 'text' , '')    
    getattr(interface.find('./oper-status'), 'text', None)
    getattr(interface.find('./mtu'), 'text', None)
    interface.find('./interface-flapped').attrib['seconds'] if interface.find('./interface-flapped') is not None else None    
    getattr(interface.find('.//input-pps'), 'text' , '')
    getattr(interface.find('.//output-pps'), 'text' , '')
```    
The `getattr(object, attribute, default)` returns the value of a specified attribute if it exists. If it does not exist, a default value is retured.

Short example where we put the collected information into a dictionary:
```
from collections import OrderedDict # ensure the ordering of the dict key/values

all_interfaces = rpc.xpath('//physical-interface|//physical-interface/logical-interface')
interfaces_list_of_dict = []

for interface in all_interfaces:
    interfaces_list_of_dict.append(
        OrderedDict({
        'interface-name' : getattr(interface.find('./name'), 'text' , ''),
        'interface-description' : getattr(interface.find('./description'), 'text' , ''),
        'interface-status' : getattr(interface.find('./oper-status'), 'text', None),
        'interface-mtu' : getattr(interface.find('./mtu'), 'text', None),
        'interface-last-flapped' : interface.find('./interface-flapped').attrib['seconds'] if interface.find('./interface-flapped') is not None else None,
        'interface-input-pps' : getattr(interface.find('.//input-pps'), 'text' , ''),
        'interface-output-pps' : getattr(interface.find('.//output-pps'), 'text' , ''),

    })
    )
```

To look at what was gathered, use json.dumps because otherwise OrderedDicts look silly:
```
from json import dumps
print(dumps(interfaces_list_of_dict, indent=4))
```

There are a lot of 'uninteresting' interfaces that Juniper has for internal use. Filtering them out is easy:
```
uninteresting = ["jsrv", "local", "igb", "ixlv", "bme", "32768", "16384", "32767"]

for interface in all_interfaces:
    if any(s in getattr(interface.find('./name'), 'text' , '') for s in uninteresting):
        continue
    if len(interface.xpath('./address-family/interface-address/ifa-local/text()')):
        getattr(interface.find('./name'), 'text' , '')
        interface.xpath('./address-family/interface-address/ifa-local/text()')
```

Note,

The code was ran against a few different Junos versions:
- QFX 15.1X53-D65.3
- MX 16.1R3-S8     
- MX10003 17.4R2-S11

I used Python 3.8.3 and the following package versions:
```   
junos-eznc            2.5.4    
lxml                  4.6.2    
```