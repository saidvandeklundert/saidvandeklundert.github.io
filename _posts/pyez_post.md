Intro
=====

Using Python and working with the Junos OS API has always been immensely satisfying. 

Most people that start out working with Junos OS using PyEZ seem to get stuck trying to figure out how to retrieve information. Since I always learn the most from short examples that I can reverse engineer or alter to fit my needs, I aim to provide you with just that. 

In this article, I will first retrieve OSPF information from a single device running Junos OS using PyEZ. In the example, I will use both the `findall` as well as the `find` XPath. The reason for this is that these XPath expressions will cover most situations. Iterating a list of BGP sessions and extracting information from individual sessions will work similar to the example I will walk you through here. Same thing for interfaces, line-cards, LDP sessions, and so on.

To wrap things up, I will also give an example on how to get the information from multiple devices.


Retrieving OSPF information
===========================

The information we are after in this example is the neighbor id, neighbor address, interface and neighbor adjacency-time. We can obtain this information by issuing the `show ospf neighbor extensive` command. To figure out what RPC we need, we simply issue `show ospf neighbor extensive |display xml rpc`:

```
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/15.1F4/junos">
    <rpc>
        <get-ospf-neighbor-information>
                <extensive/>
        </get-ospf-neighbor-information>
    </rpc>
..
</rpc-reply>
```

What is enclosed in the rpc tag will translate to `get_ospf_neighbor_information(extensive=True)` in your Python script. 

To figure out what to extract the data from the return output, we issue the ` show ospf neighbor extensive |display xml` command:

```
said@ar01.ams> show ospf neighbor extensive |display xml    
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/15.1F4/junos">
    <ospf-neighbor-information xmlns="http://xml.juniper.net/junos/15.1F4/junos-routing">
        <ospf-neighbor>
            <neighbor-address>10.253.158.131</neighbor-address>
            <interface-name>ae11.0</interface-name>
            ..
            <neighbor-id>10.253.158.254</neighbor-id>
            ..
            <neighbor-adjacency-time junos:seconds="65269304">
                107w6d 10:21:44
            </neighbor-adjacency-time>
            ..
        </ospf-neighbor>
        <ospf-neighbor>
            <neighbor-address>10.253.158.149</neighbor-address>
            <interface-name>ae12.0</interface-name>
            ..
            <neighbor-id>10.253.158.253</neighbor-id>
            ..
            <neighbor-adjacency-time junos:seconds="65183944">
                107w5d 10:39:04
            </neighbor-adjacency-time>
            ..
        </ospf-neighbor>
        ..
</rpc-reply>
```

From this output, we can see that the text-nodes we are looking for are contained in the following element nodes:
- neighbor-id
- neighbor-address
- interface-name
- neighbor-adjacency-time

Actually, for the neighbor-adjacency-time, the attribute node (`junos:seconds="65183944"`) is more interesting, so we'll grab that on instead.

We want to retrieve this information for every adjacency and we need to return the information in a way that we can use it later on. We will start off with a function that collects and returns the relevant information for 1 device and have it do the following:
- Log into the node
- Issue the RPC
- Iterate all the OSPF adjacencies
- Extract the information for every adjacency that is found
- Return this information in a dictionary

The example I came up with is the following:

```python
def jun_ospf_neighbor_extensive(username, pwd, host ):
    
    return_dict = {}

    dev = Device(host=host, user=username, password=pwd)

    dev.open()    
    ospf_information = dev.rpc.get_ospf_neighbor_information(extensive=True)
    dev.close()

    ospf_neighbors = ospf_information.findall('.//ospf-neighbor')

    for neighbor in ospf_neighbors:
        neighbor_id = neighbor.find('.//neighbor-id').text
        address = neighbor.find('.//neighbor-address').text
        interface = neighbor.find('.//interface-name').text
        uptime = neighbor.find('.//neighbor-adjacency-time').attrib['seconds']
        return_dict[interface] = { 
            'neighbor-id' : neighbor_id,
            'neighbor-address' : address,
            'interface-name' : interface,
            'neighbor-adjacency-time' : uptime,
        }
        
    return return_dict
```

Let’s break this down and describe what is happening.

The following is used to open a connection to the device, issue the RPC, store the device response in the `ospf_information` variable and close the connection:

```python
    dev.open()    
    ospf_information = dev.rpc.get_ospf_neighbor_information(extensive=True)
    dev.close()
```

The information that is now stored in `ospf_information` equates to the entire output of the `show ospf extensive | display xml` command. What we need to do is iterate all the ospf neighbors so that we can retrieve information for every individual neighbor. To this end, we turn to the `findall` method:

```python
ospf_neighbors = ospf_information.findall('.//ospf-neighbor')
```

The `findall` method is used to return a list of matching elements. In this case, the matching element is the `ospf-neighbor`. 

The list that `findall` returns is stored in `ospf_neighbors`. If we wanted to see how this information looks, we could decide to use `etree.tostring`. 

We can do this by adding the following import `from lxml import etree`. Additionaly, we would add the following to the function right after using `findall`:

```python
    for neighbor in ospf_neighbors:
        print(etree.tostring(neighbor, pretty_print=True)) 
```

When we run the function after adding this, this will print every item in the list to screen. The output should look something like this:

```
<ospf-neighbor>
<neighbor-address>10.97.18.248</neighbor-address>
<interface-name>ae6.0</interface-name>
..
<neighbor-id>10.45.16.30</neighbor-id>
..
<neighbor-adjacency-time seconds="24429948">
40w2d 18:05:48
</neighbor-adjacency-time>
..
</ospf-neighbor>


<ospf-neighbor>
<neighbor-address>10.253.158.129</neighbor-address>
<interface-name>ae7.0</interface-name>
..
<neighbor-id>10.253.158.252</neighbor-id>
..
<neighbor-adjacency-time seconds="92141820">
152w2d 10:57:00
</neighbor-adjacency-time>
..
</ospf-neighbor>
```

From the complete XML that was returned by the Juniper device, we managed to extract a list of XML objects that contain information on individual OSPF neighbors. We can iterate this list and search every individual OSPF neighbor for the things we are after using the `find` method:

```python
    for neighbor in ospf_neighbors:
        neighbor_id = neighbor.find('.//neighbor-id').text
        address = neighbor.find('.//neighbor-address').text
        interface = neighbor.find('.//interface-name').text
        uptime = neighbor.find('.//neighbor-adjacency-time').attrib['seconds']
```

We store the information in a variable. When we check the uptime, we do not just grab the text node. Here, we retrieve the attribute node because it is easier to work with `seconds` instead of some string.

The last part of the function is storing these values in a dictionary. We instantiated that dictionary a little earlier when we used `return_dict = {}` in the beginning of the function. Now, while we are still inside the for loop, we store the variables in that dictionary like so:

```python
        return_dict[interface] = { 
            'neighbor-address' : address,
            'interface-name' : interface,
            'neighbor-adjacency-time' : uptime,
        }
```

The  last `return return_dict` statement returns the dictionary for future use.

An easy way to run this function in a script would be the following:

```python
from jnpr.junos import Device
from pprint import pprint
from lxml import etree

def jun_ospf_neighbor_extensive(username, pwd, host ):

    return_dict = {}

    dev = Device(host=host, user=username, password=pwd)

    dev.open()
    ospf_information = dev.rpc.get_ospf_neighbor_information(extensive=True)
    dev.close()

    ospf_neighbors = ospf_information.findall('.//ospf-neighbor')

    for neighbor in ospf_neighbors:
        #print(etree.tostring(neighbor, pretty_print=True)) 
        neighbor_id = neighbor.find('.//neighbor-id').text
        address = neighbor.find('.//neighbor-address').text
        interface = neighbor.find('.//interface-name').text
        uptime = neighbor.find('.//neighbor-adjacency-time').attrib['seconds']
        return_dict[interface] = { 
            'neighbor-id' : neighbor_id,
            'neighbor-address' : address,
            'interface-name' : interface,
            'neighbor-adjacency-time' : uptime,
        }

    return return_dict

if __name__ == "__main__":
    # to run the function against a single host
    import sys
    host = str(sys.argv[1])
    pprint(jun_ospf_neighbor_extensive('username_123', 'password_123', host ))
```

Because we are using `sys.argv`, we can target individual devices when we run the script like so:

```bash
[said@server]$ python get_ospf.py ar01.ams
{'ae11.0': {'interface-name': 'ae11.0',
            'neighbor-address': '10.253.158.131',
            'neighbor-adjacency-time': '66251082',
            'neighbor-id': '10.253.158.254'},
 'ae12.0': {'interface-name': 'ae12.0',
            'neighbor-address': '10.253.158.149',
            'neighbor-adjacency-time': '66165722',
            'neighbor-id': '10.253.158.253'},
 'ae4.0': {'interface-name': 'ae4.0',
           'neighbor-address': '10.253.158.135',
           'neighbor-adjacency-time': '18649136',
           'neighbor-id': '10.253.158.250'},
 'ae7.0': {'interface-name': 'ae7.0',
           'neighbor-address': '10.253.158.129',
           'neighbor-adjacency-time': '93122299',
           'neighbor-id': '10.253.158.252'}}
```
 

Checking multiple nodes
=======================

Let’s continue to work on what we have done so far and make our script suitable to collect information from multiple devices. The following function will make it easy to iterate a list of devices, call the function we have made earlier and store the output in a single dictionary:

```python
def jun_ospf_neighbor_extensive_network(username, pwd, hosts = []):
    network_ospf_dict = {}
    
    for host in hosts:
        ospf_dict = jun_ospf_neighbor_extensive(username, pwd, host )
        network_ospf_dict[host] = ospf_dict

    return network_ospf_dict
```

This function takes in a username, password and a list (of hosts). It starts by instatiating a dictionary called `network_ospf_dict`. After that, it will iterate the list of hosts. For every host in the list, it will run `jun_ospf_neighbor_extensive`. The returned output is stored in the `network_ospf_dict` which is returned in the end.

Let's clean up all references to `etree`, add the new function and change the `__main__`. We now have the following script:

```python
from jnpr.junos import Device
from pprint import pprint

def jun_ospf_neighbor_extensive(username, pwd, host ):

    return_dict = {}

    dev = Device(host=host, user=username, password=pwd)

    dev.open()
    ospf_information = dev.rpc.get_ospf_neighbor_information(extensive=True)
    dev.close()

    ospf_neighbors = ospf_information.findall('.//ospf-neighbor')

    for neighbor in ospf_neighbors:
        neighbor_id = neighbor.find('.//neighbor-id').text
        address = neighbor.find('.//neighbor-address').text
        interface = neighbor.find('.//interface-name').text
        uptime = neighbor.find('.//neighbor-adjacency-time').attrib['seconds']
        return_dict[interface] = { 
            'neighbor-id' : neighbor_id,
            'neighbor-address' : address,
            'interface-name' : interface,
            'neighbor-adjacency-time' : uptime,
        }

    return return_dict

def jun_ospf_neighbor_extensive_network(username, pwd, hosts = []):
    network_ospf_dict = {}
    
    for host in hosts:
        ospf_dict = jun_ospf_neighbor_extensive(username, pwd, host )
        network_ospf_dict[host] = ospf_dict

    return network_ospf_dict


if __name__ == "__main__":
    hosts = ['ar03.ams', 'pr02.lon04', ]
    pprint(jun_ospf_neighbor_extensive_network('username_123', 'password_123', hosts))
```

When we run it against two hosts, we get the following result:

```bash
{'ar03.ams': {'ae11.0': {'interface-name': 'ae11.0',
                            'neighbor-address': '10.8.198.131',
                            'neighbor-adjacency-time': '74643455',
                            'neighbor-id': '10.8.198.254'},
                 'ae12.0': {'interface-name': 'ae12.0',
                            'neighbor-address': '10.8.198.135',
                            'neighbor-adjacency-time': '7350164',
                            'neighbor-id': '10.8.198.253'},
                 'ae8.0': {'interface-name': 'ae8.0',
                           'neighbor-address': '10.8.198.141',
                           'neighbor-adjacency-time': '74643488',
                           'neighbor-id': '10.8.198.250'},
                 'ae9.0': {'interface-name': 'ae9.0',
                           'neighbor-address': '10.8.198.143',
                           'neighbor-adjacency-time': '74643490',
                           'neighbor-id': '10.8.198.249'}},
 'pr02.lon04': {'ae0.12': {'interface-name': 'ae0.12',
                           'neighbor-address': '192.254.3.84',
                           'neighbor-adjacency-time': '41364841',
                           'neighbor-id': '10.0.140.234'},
                 'ae1.12': {'interface-name': 'ae1.12',
                           'neighbor-address': '192.254.4.84',
                           'neighbor-adjacency-time': '41364843',
                           'neighbor-id': '10.0.140.235'},
                 'ae101.0': {'interface-name': 'ae101.0',
                             'neighbor-address': '192.254.35.71',
                             'neighbor-adjacency-time': '32976830',
                             'neighbor-id': '10.0.140.236'},
                 'ae42.0': {'interface-name': 'ae42.0',
                            'neighbor-address': '192.254.9.123',
                            'neighbor-adjacency-time': '40529384',
                            'neighbor-id': '10.0.141.255'}}}
```


Wrapping up
===========

We wrote a function that retrieves OSPF information by talking to the Juniper API. From the XML response, we used XPATH expressions to retrieve the information we want. 

First we used `findall`. This gave us a list with information on individual OSPF neighbors. After this, we used `find` to obtain the exact information we needed from every individual neighbor. 

After obtaining the information for 1 device, we then created something that can iterate a list of devices and store everything in 1 dictionary. This way, you will be able to get information on devices for the entire network.


