---
layout: post
title: Screen scraping basics for network engineers
tags: [automation, python, napalm, netmiko ]
image: /img/python-logo.jpg
---

This article contains several examples I could have used after reading up on the basics in Python. After I read the first chapters of <b>Automate the Boring Stuff with Python</b> and <b>Learning Python, 5th Edition</b>, I struggled to put the concepts I read about into practice. I understood the basic Python data structures and string methods but I had a hard time applying it anywhere.

As a network engineer, I was very familiar with the CLI. I found that it was a lot of fun putting the stuff I learned into practice by working with CLI output. In this article, I will start by giving a few basic examples on how to send a command to a device and retrieve the output as a string. After this, I will give a few examples on how to break the strings down into smaller parts and find what you are looking for.


Retrieving device output:
=========================

There are numerous ways on how to retrieve command output from a device. The following are 2 commonly used libraries that are quite popular.

### Netmiko:

Netmiko is a <b>Multi-vendor library to simplify Paramiko SSH connections to network devices</b>. <a href="http://www.paramiko.org/" target="_blank">Paramiko</a> is a Python implementation of the SSHv2 protocol which can be difficult to use. Netmiko hides that complexity for us, which makes sending a command to a device and retrieving the output very easy.

The following is an example of a script that sends the <b>show version</b> command to a device running NX-OS:

```python
from netmiko import ConnectHandler
import getpass

password = getpass.getpass() 

login_info = {
    'device_type': 'cisco_nxos',
    'host':   '10.10.10.254',
    'username': 'lab',
    'password': password,
}

net_connect = ConnectHandler(**login_info)

s = net_connect.send_command('show version')

net_connect.disconnect()

print(s)
```

What happens here is the following:
- netmiko and getpass are imported
- after the imports, getpass is used to ask for a password
- the 'login_info' dictionary that contains login information is created
- we pass the login info to the <b>ConnectHandler</b> and setup a connection to the device
- with <b>net_connect.send_command('show version')</b> we send the command to the device and store the return in the variable called <b>s</b>
- we disconnect from the device
- we print the result to screen

When we run the above script, we see the following:

<pre style="font-size:12px">
< output omitted >

Hardware
  cisco Nexus7700 C7718 (18 Slot) Chassis ("Supervisor Module-2")
  Intel(R) Xeon(R) CPU         with 32939304 kB of memory.
  Processor Board ID JAE2036044P

  Device name: lab.ams01
  bootflash:    4014080 kB
  slot0:       15769534 kB (expansion flash)

Kernel uptime is 508 day(s), 11 hour(s), 37 minute(s), 35 second(s)

< output omitted >
</pre>

In case you are working with other vendors and/or devices, you have to pass the appropriate value as the <b>device_type</b> in the 'login_info' dictionary. Here are some of the usual suspects I use the most:

  |                     | Arista EOS        | Junos         | Cisco IOS-XR     | Cisco NX-OS      | Cisco IOS
  | ------------------- | ----------------- | ------------- | ---------------- | ---------------- | --------- 
  | **device_type**     | arista_eos        | juniper       | cisco_xr         | cisco_nxos       | cisco_ios

The author of the package has his own 'getting started' <a href="https://pynet.twb-tech.com/blog/automation/netmiko.html" target="_blank">here</a>.

### NAPALM:

The following is an example script on how to issue a command using NAPALM:

```python
import napalm
import getpass

password = getpass.getpass()  

driver = napalm.get_network_driver('iosxr')
device = driver(hostname='10.50.169.101', username='lab', password=password)
device.open()
return_dictionary = device.cli(['show ospf neighbor', ])
device.close()
s = return_dictionary['show ospf neighbor']
print(s)
```

In the previous example:
- NAPALM and getpass are imported
- after the imports, the user is asked for a password using getpass
- the driver that details what type of device to connect to is selected
- the device object is created
- a connection to the device is opened
- the cli method is used to send a list of commands to the device (in this case, only 1 command: <b>show ospf neighbor</b>)
- the connection to the device is closed
- we retrieve the output of the command from the dictionary that is returned by 'device.cli' and store it in the variable called <b>s</b>
- we print <b>s</b> to screen


The scripts outputs the following:
<pre style="font-size:12px">
* Indicates MADJ interface
# Indicates Neighbor awaiting BFD session up

Neighbors for OSPF 10

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.0.19.237     128   FULL/  -        00:00:39    10.254.63.12   HundredGigE0/0/0/4
    Neighbor is up for 5w5d
10.0.19.238     128   FULL/  -        00:00:39    10.254.63.14   HundredGigE0/0/0/5
    Neighbor is up for 5w5d

Total neighbor count: 2

</pre>
For more on NAPALM and wat drivers to select, check out <a href="https://saidvandeklundert.net/2019-09-20-napalm/" target="_blank">this</a> post.

In the rest of the article, I will leave out the part where I retrieve the device output. Instead of showing the way I use Netmiko or NAPALM, I will just put in <b>s = xxxx</b> to detail what the string is that I am working with.


Breaking it down using splitlines and split:
============================================

Instead of working with large multiline strings that are returned by a device, it is easier to work with the individual lines. The splitlines string method turns a single string into a list of strings where every line is an item in that list.

Example:

```python
s = """This is line 1.
This is line 2.
This is line3."""
s.splitlines()
```

This would output the following:

<pre style="font-size:12px">
['This is line 1.', 'This is line 2.', 'This is line3.']
</pre>

The same logic can be applied to a single line using <b>split</b>. The <b>split</b> method allows you to turn a line into a list of items where every item is a word:

```python
s = 'Use more string methods, less regex.'
s.split()
```

The above gives us:

<pre style="font-size:12px">
['Use', 'more', 'string', 'methods,', 'less', 'regex.']
</pre>

To be more precise, it will split a string using a whitespace as separator. You can use something other than this default value like so:

```python
s = '192.168.1.1/24'
s.split('/')        
['192.168.1.1', '24']
```

Combining split and splitlines can sometimes be enough to extract a value without using regex. Let's look at the following example where we extract the software version an Arista device is running:

```python
s = """
Arista DCS-7050TX-64-R
Hardware version:    21.12
Serial number:       SSJ17433414
System MAC address:  7483.ef2c.bd81

Software image version: 4.20.14M
Architecture:           i386
Internal build version: 4.20.14M-12819260.42014M
Internal build ID:      ab77e866-fa99-4d12-a4fe-d7eb87b90f5c

Uptime:                 12 weeks, 2 days, 15 hours and 56 minutes
Total memory:           3818208 kB
Free memory:            2640048 kB
"""

for line in s.splitlines():                           # 1. 's' becomes a list of strings
    if 'Software' in line:                            # 2. stop at line 'Software image version: 4.20.14M'
        list_of_chunks = line.split(':')              # 3. ['Software image version', ' 4.20.14M']
        software_version = list_of_chunks[1]          # 4. ' 4.20.14M'
        software_version = software_version.strip()   # 5. '4.20.14M'

print(software_version)
```

In the comments, I am explaining the different things that are happening when the code is executed:

1. Here we split <b>s</b> into a list of strings and we step through them one by one.
2. We check every line for the presence of the word 'Software'. When found, the if block is executed.
3. We split the line into a list of strings on the ':' character. The <b>list_of_chunks</b> list now contains <b>['Software image version', ' 4.20.14M']</b>.
4. The second item in the list is assigned to the 'software_version' variable, which contains the <b>' 4.20.14M'</b> string.
5. We strip the string of any leading and trailing whitespaces, so we end up with <b>'4.20.14M'</b>.

Running the above code would output the following:

<pre style="font-size:12px">
4.20.14M
</pre>

We could have been concise as well by calling all required methods on <b>line</b>, like so:

```python
for line in s.splitlines():
    if 'Software' in line:
        print(line.split(':')[1].strip())
```

Let's look into another example on a Cisco NX-OS. The string we are working with is the output of the <b>show ipv6 ospfv3 neighbors</b> command. To extract the OSPFv3 neighbor ID and interface behind which we find the neighbor, we can use the same approach as we used earlier:

```python
from pprint import pprint

s = """
 OSPFv3 Process ID 20 VRF default
 Total number of neighbors: 5
 Neighbor ID     Pri State            Up Time  Interface ID    Interface
 10.168.118.254   1 FULL/ -          1y20w    6               Vlan2 
   Neighbor address fe80::2de:fbff:fed2:f241
 10.168.118.241 128 FULL/ -          1y2w     7               Po1 
   Neighbor address fe80::7e25:86ff:fef6:b029
 10.168.118.242 128 FULL/ -          1y19w    7               Po2 
   Neighbor address fe80::7e25:86ff:fef5:dd88
 10.168.118.243 128 FULL/ -          1y19w    7               Po3 
   Neighbor address fe80::7e25:86ff:feee:d6d2
 10.168.118.244 128 FULL/ -          1y19w    7               Po4 
   Neighbor address fe80::7e25:86ff:fef5:cd88
"""

ospf_neighbor_d = {}

for line in s.splitlines():
  if 'FULL' in line:
    ospf_neighbor_id = line.split()[0]
    ospf_neighbor_int = line.split()[-1]
    print('{} sits behind interface {}'.format(ospf_neighbor_id, ospf_neighbor_int))
    ospf_neighbor_d[ospf_neighbor_int] = ospf_neighbor_id

pprint(ospf_neighbor_d)
```

In the above example, we start out creating a dictionary for later use. After this, we iterate the lines of the string. 

For every line that has 'FULL' in it, we split the string and take the first and last word which we assign to the variables <b>ospf_neighbor_id</b> and <b>ospf_neighbor_int</b>.

We use the variables to print a message to the terminal and we build a dictionary where the OSPF neighbor interface is the key and the OSPF neighbor ID is the value.

The example code would give us the following:

<pre style="font-size:12px">
10.168.118.254 sits behind interface Vlan2
10.168.118.241 sits behind interface Po1
10.168.118.242 sits behind interface Po2
10.168.118.243 sits behind interface Po3
10.168.118.244 sits behind interface Po4
{'Po1': '10.168.118.241',
 'Po2': '10.168.118.242',
 'Po3': '10.168.118.243',
 'Po4': '10.168.118.244',
 'Vlan2': '10.168.118.254'}
</pre>



Using any and all to find what you are looking for:
===================================================

Quite often, you will be looking for multiple values. Let's have a look at the following string:

```python
s = """
router bgp 65500 neighbor 10.0.19.1 remote-as 65501
router bgp 65500 neighbor 10.0.19.1 update-source Loopback0
router bgp 65500 neighbor 10.0.19.1 address-family ipv4 unicast 
router bgp 65500 neighbor 10.0.19.1 address-family ipv4 unicast next-hop-self
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast 
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-import-policy in
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-export-policy out

set protocols bgp group exchange type external
set protocols bgp group exchange import exchange-import
set protocols bgp group exchange export exchange-export
set protocols bgp group exchange neighbor 10.0.0.1 family inet unicast
set protocols bgp group exchange neighbor 10.0.0.1 export deny-all
set protocols bgp group exchange neighbor 10.0.0.1 peer-as 65500
set protocols bgp group exchange neighbor 2001:DB8::1 family inet6 unicast
set protocols bgp group exchange neighbor 2001:DB8::1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 peer-as 65500
"""
```

The string is a snippet of an IOS-XR configuration produced using <b>show running-config formal</b> and a Juniper configuration produced using <b>show configuration | display set</b>. Let's say we want to find all the configuration lines that have anything in them related to BGP export or import policy configuration.

What we could do is something like this:

```python
for line in s.splitlines():
  if 'export' in line or 'import' in line or 'route-policy' in line:
    print(line)
```

This would produce the following:

<pre style="font-size:12px">
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-import-policy in
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-export-policy out
set protocols bgp group exchange import exchange-import
set protocols bgp group exchange export exchange-export
set protocols bgp group exchange neighbor 10.0.0.1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all
</pre>

It works, but it does not look very nice. Also, consider the abomination when you want to look for 5 or more items in a string:

```python
if 'export' in line or 'import' in line or 'route-policy' in line or 'something'  in line or 'something else' in line or 'another thing' in line:
```

Using <b>any()</b> will 'return True if any element of the iterable is true'. This allows us to specify the items we are interested in, as a list:

```python
interesting_items = [ 'export', 'import',  ]

for line in s.splitlines():
  if any(x in line for x in interesting_items):
    print(line)
```

This produces the same output as before:

<pre style="font-size:12px">
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-import-policy in
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-export-policy out
set protocols bgp group exchange import exchange-import
set protocols bgp group exchange export exchange-export
set protocols bgp group exchange neighbor 10.0.0.1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all
</pre>


In case you are looking for multiple items that have to be present in a string, use the <b>all()</b> method. This will 'return True if all elements of the iterable are true'. In the following example, we will print the line in case all the words in the list are found:

```python
interesting_items = [ 'exchange', 'import',  ]

for line in s.splitlines():
  if all(x in line for x in interesting_items):
    print(line)
```

Using the same string as before, this will produce the following result:

<pre style="font-size:12px">
set protocols bgp group exchange import exchange-import
set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all
</pre>


Closing thoughts
================

Learning Python is a lot of fun. After reading up on the basics, I recommend you try and practice on information retrieval by doing some screen scraping. You do not need to write a lot of Python and you can start doing useful things in no time. I hope the examples in this post help you.