---
layout: post
title: Navigating Juniper RPC returns using XPATH
tags: [ python, automation, juniper, pyez, ]
image: /img/juniper_logo.jpg
---

Many people dread XML and would rather work with JSON. However, XPATH can be extremely powerful when dealing with the Juniper XML API.

Even though the Juniper API can be made to return JSON instead of XML, I prefer sticking to XML. With little code, I tend to be able to find exactly what I am looking for.

This blog contains my commonly used snippets of code that retrieve and navigate information from the Juniper XML API using XPATH.


## Code that was used to execute the RPC:

The data I will be working with in the article is retrieved using the following code:

<pre style="font-size:12px">
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
</pre>

## How the RPC was mapped from the Junos CLI command:

The `rpc` that is referred to in the rest of the examples is the device response to `get_interface_information(extensive=True)`. This is the RPC to retrieve the information that can be displayed with the `show interfaces extensive` command:

```xml
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

I stored the example script as `example.py` and ran it with `python -i`. This will drop you in the interpreter and allows you to paste in the following snippets and examples.

## Examples on how to use XPATH to navigate the RPC return:

XPATH getting all the interface names:
<pre style="font-size:12px">
rpc.xpath('//physical-interface/name/text()')
</pre>

XPATH getting all the logical interface names:
<pre style="font-size:12px">
rpc.xpath('//physical-interface/logical-interface/name/text()')
</pre>

XPATH getting all the interface names of the physical as well as the logical interfaces:
<pre style="font-size:12px">
rpc.xpath('//physical-interface/name/text()|//physical-interface/logical-interface/name/text()')
</pre>

XPATH getting all the physical and logical interface XML data:
<pre style="font-size:12px">
rpc.xpath('//physical-interface|//physical-interface/logical-interface')
</pre>

Using XPATH, we slice the XML into a list of items. To see the content of the items, that we could do another XPATH expression on, we can use the following:
<pre style="font-size:12px">
for interface_xml in rpc.xpath('//physical-interface|//physical-interface/logical-interface'):    
    print(etree.tostring(interface_xml))
</pre>

That works, but it does not look very pretty. A (somewhat involved) way to make it look pretty:
<pre style="font-size:12px">
import xml.dom.minidom
for interface_xml in rpc.xpath('//physical-interface|//physical-interface/logical-interface'):
    xml_minidom = xml.dom.minidom.parseString(etree.tostring(interface_xml))
    interface_xml_pretty = xml_minidom.toprettyxml()
    print(interface_xml_pretty)
</pre>

Even easier then using the above would be to just check the XML on the device.

XPATH getting the physical interface names in case they include 'et':
<pre style="font-size:12px">
rpc.xpath('physical-interface[contains(name, "et-")]/name/text()')
</pre>

XPATH getting the logical interface names in case they include 'irb':
<pre style="font-size:12px">
rpc.xpath('physical-interface/logical-interface[contains(name, "irb")]/name/text()') 
</pre>

Using regular expresions in lxml:
<pre style="font-size:12px">
ns = {"re": "http://exslt.org/regular-expressions"}
# RE to get 100G:
rpc.xpath('physical-interface[re:match(name, "et")]/name/text()', namespaces=ns) 

# RE to get 100G on line card position 1 and 2:
rpc.xpath('physical-interface[re:match(name, "et-[1,2]")]/name/text()', namespaces=ns)

# RE to get 100G on line card position 1 and 2 while ignoring upper and lower case:
rpc.xpath('physical-interface[re:match(name, "ET-[1,2]", "i")]/name/text()', namespaces=ns)
</pre>

XPATH to get all interfaces, physical as well as logical:
<pre style="font-size:12px">
all_interfaces = rpc.xpath('//physical-interface|//physical-interface/logical-interface')
</pre>

Now we can iterate `all_interfaces` to get the name of the interfaces using the `find` method:
<pre style="font-size:12px">
for interface in all_interfaces:
    # extract the name using find:
    interface.find('./name').text if interface.find('./name') is not None else None
</pre>

The `if interface.find('./name') is not None else None` was added to avoid running into exceptions.

Alternatively, we use an XPATH to accomplish the same thing again:
<pre style="font-size:12px">
for interface in all_interfaces:
    # extract the name using XPATH:
    interface.xpath('./name/text()') 
</pre>

We can also inspect the text of all the child nodes, just to see what is on offer:
<pre style="font-size:12px">
for interface in all_interfaces:
    interface.xpath('child::*/text()')
    interface.xpath('child::*/text()')
</pre>

Sometimes, the attributes offer more interesting data. To access these attributes, we can use the following:
<pre style="font-size:12px">
for interface in all_interfaces:
    interface.find('./interface-flapped').attrib['seconds'] if interface.find('./interface-flapped') is not None else None
</pre>

Safely accessing attributes of can also be done using the Python built-in `getattr`:
<pre style="font-size:12px">
all_interfaces = rpc.xpath('//physical-interface|//physical-interface/logical-interface')
for interface in all_interfaces:
    getattr(interface.find('./name'), 'text' , '')
    getattr(interface.find('./description'), 'text' , '')    
    getattr(interface.find('./oper-status'), 'text', None)
    getattr(interface.find('./mtu'), 'text', None)
    interface.find('./interface-flapped').attrib['seconds'] if interface.find('./interface-flapped') is not None else None    
    getattr(interface.find('.//input-pps'), 'text' , '')
    getattr(interface.find('.//output-pps'), 'text' , '')
</pre>

The `getattr(object, attribute, default)` returns the value of a specified attribute if it exists. If it does not exist, a default value is returned.

Short example where we put the collected information into a dictionary:
<pre style="font-size:12px">
# to ensure ordering of dict key/values remains the same, no needed in python 3.9+
from collections import OrderedDict 

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
</pre>

To look at what was gathered, use json.dumps because otherwise OrderedDicts look silly:
<pre style="font-size:12px">
from json import dumps
print(dumps(interfaces_list_of_dict, indent=4))
</pre>

There are a lot of 'uninteresting' interfaces that Juniper has for internal use. Filtering them out is easy:
<pre style="font-size:12px">
uninteresting = ["jsrv", "local", "igb", "ixlv", "bme", "32768", "16384", "32767"]

for interface in all_interfaces:
    if any(s in getattr(interface.find('./name'), 'text' , '') for s in uninteresting):
        continue
    if len(interface.xpath('./address-family/interface-address/ifa-local/text()')):
        getattr(interface.find('./name'), 'text' , '')
        interface.xpath('./address-family/interface-address/ifa-local/text()')
</pre>

I have recorded some sample output right [here](https://github.com/saidvandeklundert/juniper/blob/master/xpath_examples.py) and [here](https://github.com/saidvandeklundert/juniper/blob/master/xpath_re_example.py).

More examples and explanation on how to work with lxml can be found [here](https://lxml.de/xpathxslt.html).

Note,

Python 3.8.3 was used with the following package versions:
```   
junos-eznc            2.5.4    
lxml                  4.6.2    
```

The Juniper devices I used were the following:

```
QFX 15.1X53-D65.3
MX 16.1R3-S8     
MX10003 17.4R2-S11
```