Intro
=====

Using Python and working with the Junos OS API has always been immensely satisfying. 

Most people that start out working with Junos OS using PyEZ seem to get stuck trying to figure out how to retrieve information for a list of devices they are interested in. They know what they want, but struggle getting started. 

Since I always learn the most from short examples that I can reverse engineer or alter to fit my needs, I aim to provide you with just that. In this article, I will first retrieve OSPF information from a single device running Junos OS using PyEZ. After this, I will move on to retrieving the information from multiple devices.
 

Retrieving OSPF information
===========================

In this example, we will be looking for the neighbor address, neighbor id, interface and neighbor adjacency-time. 

We know this information is revealed when we issue `show ospf neighbor extensive` command. To figure out what RPC we need, we simple issue `show ospf neighbor extensive |display xml rpc` which will give us the following:

```
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/15.1F4/junos">
    <rpc>
        <get-ospf-neighbor-information>
                <extensive/>
        </get-ospf-neighbor-information>
    </rpc>
    <cli>
        <banner>{master}</banner>
    </cli>
</rpc-reply>
```

This will translate to `get_ospf_neighbor_information(extensive=True)` in your Python script. 

To figure out how to extract the data from the return output, we issue the ` show ospf neighbor extensive |display xml` command:

```
said@ar01.ams> show ospf neighbor extensive |display xml    
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/15.1F4/junos">
    <ospf-neighbor-information xmlns="http://xml.juniper.net/junos/15.1F4/junos-routing">
        <ospf-neighbor>
            <neighbor-address>10.253.158.131</neighbor-address>
            <interface-name>ae11.0</interface-name>
            <ospf-neighbor-state>Full</ospf-neighbor-state>
            <neighbor-id>10.253.158.254</neighbor-id>
            <neighbor-priority>1</neighbor-priority>
            <activity-timer>31</activity-timer>
            <ospf-area>0.0.0.0</ospf-area>
            <options>0x52</options>
            <dr-address>0.0.0.0</dr-address>
            <bdr-address>0.0.0.0</bdr-address>
            <neighbor-up-time junos:seconds="65269304">
                107w6d 10:21:44
            </neighbor-up-time>
            <neighbor-adjacency-time junos:seconds="65269304">
                107w6d 10:21:44
            </neighbor-adjacency-time>
            ..
            < output omitted>
            ..
        </ospf-neighbor>
        <ospf-neighbor>
            <neighbor-address>10.253.158.149</neighbor-address>
            <interface-name>ae12.0</interface-name>
            <ospf-neighbor-state>Full</ospf-neighbor-state>
            <neighbor-id>10.253.158.253</neighbor-id>
            <neighbor-priority>1</neighbor-priority>
            <activity-timer>31</activity-timer>
            <ospf-area>0.0.0.0</ospf-area>
            <options>0x52</options>
            <dr-address>0.0.0.0</dr-address>
            <bdr-address>0.0.0.0</bdr-address>
            <neighbor-up-time junos:seconds="65183945">
                107w5d 10:39:05
            </neighbor-up-time>
            <neighbor-adjacency-time junos:seconds="65183944">
                107w5d 10:39:04
            </neighbor-adjacency-time>
            ..
            < output omitted>
            ..
        </ospf-neighbor>
        ..
        < output omitted>
        ..
</rpc-reply>
```

Here we can see what fields contain the information we are looking for. The fields are:
- neighbor-id
- neighbor-address
- interface-name
- neighbor-adjacency-time

We want to retrieve this information for every adjacency and we need to return the information in a way that we can use it later on. For this reason, we will have the function return the information as a dictionary.

So starting off with a function that collects and returns the relevant information from 1 node, we will write a function that does the following:
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

The following is used to open a connection to the device, retrieve the information and store it in the ‘ospf_information’ variable and close the connection:
```python
    dev.open()    
    ospf_information = dev.rpc.get_ospf_neighbor_information(extensive=True)
    dev.close()
```

The `ospf_information` contains all the data that is returned. What we need to do is iterate all the ospf neighbors so that we can retrieve information for every individual neighbor. To this end, we turn to the `findall` method:
```python
ospf_neighbors = ospf_information.findall('.//ospf-neighbor')
```

The `findall` method is used to return a list of matching elements. In this case, the matching element is the `ospf-neighbor`. 

The list that `findall` returns is stored in `ospf_neighbors`. If we wanted to see how this information looks, we could decide to use `etree.tostring`. 
Right after the `findall`, we add two lines to the function:
```python
    for neighbor in ospf_neighbors:
        print(etree.tostring(neighbor, pretty_print=True, encoding='unicode')) 
```
When we run the function after adding this, this will print every item in the list to screen. The output should look something like this:
```
<ospf-neighbor>
<neighbor-address>10.97.18.248</neighbor-address>
<interface-name>ae6.0</interface-name>
<ospf-neighbor-state>Full</ospf-neighbor-state>
<neighbor-id>10.45.16.30</neighbor-id>
<neighbor-priority>128</neighbor-priority>
<activity-timer>34</activity-timer>
<ospf-area>0.0.0.0</ospf-area>
<options>0x52</options>
<dr-address>0.0.0.0</dr-address>
<bdr-address>0.0.0.0</bdr-address>
<neighbor-up-time seconds="24429948">
40w2d 18:05:48
</neighbor-up-time>
<neighbor-adjacency-time seconds="24429948">
40w2d 18:05:48
</neighbor-adjacency-time>
<ospf-neighbor-topology>
<ospf-topology-name>default</ospf-topology-name>
<ospf-topology-id>0</ospf-topology-id>
<ospf-neighbor-topology-state>Bidirectional</ospf-neighbor-topology-state>
</ospf-neighbor-topology>
</ospf-neighbor>


<ospf-neighbor>
<neighbor-address>10.253.158.129</neighbor-address>
<interface-name>ae7.0</interface-name>
<ospf-neighbor-state>Full</ospf-neighbor-state>
<neighbor-id>10.253.158.252</neighbor-id>
<neighbor-priority>128</neighbor-priority>
<activity-timer>38</activity-timer>
<ospf-area>0.0.0.0</ospf-area>
<options>0x52</options>
<dr-address>0.0.0.0</dr-address>
<bdr-address>0.0.0.0</bdr-address>
<neighbor-up-time seconds="92141820">
152w2d 10:57:00
</neighbor-up-time>
<neighbor-adjacency-time seconds="92141820">
152w2d 10:57:00
</neighbor-adjacency-time>
<lsa-list>
        ..
        < output omitted>
        ..
</lsa-list>
<ospf-neighbor-topology>
<ospf-topology-name>default</ospf-topology-name>
<ospf-topology-id>0</ospf-topology-id>
<ospf-neighbor-topology-state>Bidirectional</ospf-neighbor-topology-state>
</ospf-neighbor-topology>
</ospf-neighbor>
..
< output omitted>
..

```
Seeing this will help understand the next part where we extract the information using the `find` method:
```python
    for neighbor in ospf_neighbors:
        neighbor_id = neighbor.find('.//neighbor-id').text
        address = neighbor.find('.//neighbor-address').text
        interface = neighbor.find('.//interface-name').text
        uptime = neighbor.find('.//neighbor-adjacency-time').attrib['seconds']
```
Here, we iterate the objects in the list and retrieve information from the fields we are interested in. 

Note that when we check the uptime, we do not just grab the text. Instead we retrieve the `seconds`.

Let’s look at the values we extract here by returning them to screen:
```python
    for neighbor in ospf_neighbors:
        neighbor_id = neighbor.find('.//neighbor-id').text
        address = neighbor.find('.//neighbor-address').text
        interface = neighbor.find('.//interface-name').text
        uptime = neighbor.find('.//neighbor-adjacency-time').attrib['seconds']
        print(neighbor_id, address, interface, uptime)
```
When we run the function with the print statements put in, we can see the following:
```
('10.253.158.254', '10.253.158.131', 'ae11.0', '66251128')
('10.253.158.253', '10.253.158.149', 'ae12.0', '66165768')
('10.253.158.240', '10.253.158.153', 'ae13.0', '93157614')
('10.253.158.241', '10.253.158.155', 'ae14.0', '93156603')
('10.253.158.250', '10.253.158.135', 'ae4.0', '18649182')
```
The last part of the function is storing these values in a dictionary. We instantiated that dictionary a little earlier (using `return_dict = {}`) and now we put everything in that dictionary like so:
```
        return_dict[interface] = { 
            'neighbor-address' : address,
            'interface-name' : interface,
            'neighbor-adjacency-time' : uptime,
        }
```
The  last `return return_dict` statement returns it for future use.

An easy way to run this function in a script would be the following:

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

if __name__ == "__main__":
    # to run the function against a single host
    import sys
    host = str(sys.argv[1])
    pprint(jun_ospf_neighbor_extensive('username_123', 'password_123', host ))
```

When we run it, we get the following:
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
 'ae13.0': {'interface-name': 'ae13.0',
            'neighbor-address': '10.253.158.153',
            'neighbor-adjacency-time': '93157568',
            'neighbor-id': '10.253.158.240'},
 'ae14.0': {'interface-name': 'ae14.0',
            'neighbor-address': '10.253.158.155',
            'neighbor-adjacency-time': '93156557',
            'neighbor-id': '10.253.158.241'},
 'ae4.0': {'interface-name': 'ae4.0',
           'neighbor-address': '10.253.158.135',
           'neighbor-adjacency-time': '18649136',
           'neighbor-id': '10.253.158.250'},
 'ae5.0': {'interface-name': 'ae5.0',
           'neighbor-address': '192.97.18.236',
           'neighbor-adjacency-time': '25413602',
           'neighbor-id': '169.45.16.29'},
 'ae6.0': {'interface-name': 'ae6.0',
           'neighbor-address': '192.97.18.248',
           'neighbor-adjacency-time': '25410427',
           'neighbor-id': '192.45.16.30'},
 'ae7.0': {'interface-name': 'ae7.0',
           'neighbor-address': '10.253.158.129',
           'neighbor-adjacency-time': '93122299',
           'neighbor-id': '10.253.158.252'}}
```
 

Checking multiple nodes
=======================

Let’s continue to work with our previous example and add the following function:

```python
def jun_ospf_neighbor_extensive_network(username, pwd, hosts = []):
    network_ospf_dict = {}
    
    for host in hosts:
        ospf_dict = jun_ospf_neighbor_extensive(username, pwd, host )
        network_ospf_dict[host] = ospf_dict

    return network_ospf_dict
```

You simply feed the function a username, a password and a list of nodes to run `jun_ospf_neighbor_extensive` against every device in the list and store the returned information in a dictionary called `network_ospf_dict`. 

The entire script now looks like this:
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
 'pr02.lon04': {'ae0.2': {'interface-name': 'ae0.2',
                           'neighbor-address': '192.254.3.84',
                           'neighbor-adjacency-time': '41364841',
                           'neighbor-id': '10.0.140.234'},
                 'ae1.2': {'interface-name': 'ae1.2',
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

