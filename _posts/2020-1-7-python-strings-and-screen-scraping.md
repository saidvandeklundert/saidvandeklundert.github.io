---
layout: post
title: Python strings and network engineering
tags: [automation, python ]
image: /img/python-logo.jpg
---

This article contains several examples I could have really used after reading up on the basics in Python. After I read the first chapters of <b>Automate the Boring Stuff with Python</b> and <b>Learning Python, 5th Edition</b>, I somewhat struggled to put the concepts I read about into practice. I knew the basic Python data structures and string methods, but was not really sure on how to actually use any of this.

As a network engineer, I was intimately familiar with the CLI and had a lot of fun putting the stuff I learned into practice by working with CLI output. In this article, I will start by giving a few basic examples on how to send a command to a device and retrieve the output as a string. After this, I will give a few examples on how to break the strins down into smaller parts and how to find what you are looking for.


Collecting the string:
======================

There are numerous ways on how to retrieve command output from a device. The following are 2 quick and easy ways.

### Netmiko:

Netmiko is a <b>Multi-vendor library to simplify Paramiko SSH connections to network devices</b>. It makes sending a command to a device and retrieving the output very easy.

The following is an example of a script that sends the <b>show version</b> command to a device running NX-OS:

```python
from netmiko import ConnectHandler
import getpass

password = getpass.getpass() 

login_info = {
    'device_type': 'cisco_nxos',
    'host':   '169.60.118.254',
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
- a dictionary with login information is created
- we pass the login info to the `ConnectHandler` and setup a connection to the device
- with `net_connect.send_command('show version')` we send the command to the device and store the return in the variable called 's'
- we disconnect from the device
- we print the result to screen

When we run the above script, we see the following:

<pre style="font-size:12px">
sh-4.4# python3 /var/tmp/tmp.py    

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

In case you are working with other vendors and/or devices, you have to pass the appropriate value as the <b>device_type</b>. Here are some of the usual suspects I use the most:

  |                     | Arista EOS        | Junos         | Cisco IOS-XR     | Cisco NX-OS      | Cisco IOS
  | ------------------- | ----------------- | ------------- | ---------------- | ---------------- | --------- 
  | **device_type**     | arista_eos        | juniper       | cisco_xr         | cisco_nxos       | cisco_ios

The author of the package has his own 'getting started' here: https://pynet.twb-tech.com/blog/automation/netmiko.html

### NAPALM:

The following is an example script on how to issue a command using NAPALM:

```python
import napalm
import getpass

password = getpass.getpass()  

driver = napalm.get_network_driver('iosxr')
device = driver(hostname='169.50.169.101', username='lab', password=password)
device.open()
return_dictionary = device.cli(['show ospf neighbor', ])
device.close()
s = return_dictionary['show ospf neighbor']
print(s)
```

In the previous example:
- NAPALM and getpass are imported
- after the imports, again, getpass is used to ask for a password
- the driver that details what type of device to connect to is selected
- the device object is created
- a connection to the device is opened
- the cli method is used to send a list of commands to the device (in this case, only 1 command: 'show ospf neighbor')
- the connection to the device is closed
- we retrieve the output of the command from the dictionary that is returned by 'device.cli' and store it in the variable called 's'
- we print 's' to screen


The scripts outputs the following:
<pre style="font-size:12px">
* Indicates MADJ interface
# Indicates Neighbor awaiting BFD session up

Neighbors for OSPF 10

Neighbor ID     Pri   State           Dead Time   Address         Interface
10.0.19.237     128   FULL/  -        00:00:39    169.254.63.12   HundredGigE0/0/0/4
    Neighbor is up for 5w5d
10.0.19.238     128   FULL/  -        00:00:39    169.254.63.14   HundredGigE0/0/0/5
    Neighbor is up for 5w5d

Total neighbor count: 2

</pre>

In the rest of the article, I will leave out the part where I retrieve the device output. Instead of showing the way I use Netmiko or NAPALM, I will just put in `s = xxxx` to detail what the string is that I am working with.


Breaking it down using splitlines and split:
============================================

Instead of working with large multiline strings that are returned by a device, it is easier to work with the individual lines. The splitlines string method turns a single string into a list of strings where every line is an item in that list. Example:

<pre style="font-size:12px">
>>> s = """This is line 1.
... This is line 2.
... This is line3."""
>>> s.splitlines()
['This is line 1.', 'This is line 2.', 'This is line3.']
</pre>

The same logic can be applied to a single line. The <b>split</b> string method allows you to turn a line into a list of items where every item is a word:

<pre style="font-size:12px">
s = 'Use more string methods, less regex.'
s.split()
['Use', 'more', 'string', 'methods,', 'less', 'regex.']
</pre>

To be more precise, it will split a string using a whitespace as separator. You can use something other than this default value like so:

<pre style="font-size:12px">
s = '192.168.1.1/24'
s.split('/')        
['192.168.1.1', '24']
</pre>

Combining split and splitlines can sometimes be enough to extract a value without using regex. Let's look at the following example:

<pre style="font-size:12px">
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

for line in s.splitlines():
    if 'Software' in line:
        print(line.split(':')[1].strip())

</pre>

Running the above code would output the following:

<pre style="font-size:12px">
4.20.14M
</pre>


Let's look intp another example on a Cisco NX-OS. The following output is returned after issuing <b>show ipv6 ospfv3 neighbors</b>:

<pre style="font-size:12px">
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
</pre>

To extract the OSPFv3 neighbor ID and interface behind which we find the neighbor, we can use the same approach as we used earlier:

```python
ospf_neighbor_d = {}
for line in s.splitlines():
  if 'FULL' in line:
    ospf_neighbor_id = line.split()[0]
    ospf_neighbor_int = line.split()[-1]
    print('{} sits behind interface {}'.format(ospf_neighbor_id, ospf_neighbor_int))
    ospf_neighbor_d[ospf_neighbor_int] = ospf_neighbor_id

print(ospf_neighbor_d)
```

In the above example, we start out creating a dictionary for later use. After this, we iterate the lines of the string. 

For every line that has 'FULL' in it, we split the string and take the first and last word, we we store as `ospf_neighbor_id` and `ospf_neighbor_int`.

We use the variables to print a message to the terminal and we build a dictionart where the OSPF neighbor interface is the key and the OSPF neighbor ID is the value.

The example code would give us the following:

<pre style="font-size:12px">
10.168.118.254 sits behind interface Vlan2
10.168.118.241 sits behind interface Po1
10.168.118.242 sits behind interface Po2
10.168.118.243 sits behind interface Po3
10.168.118.244 sits behind interface Po4
{'Vlan2': '10.168.118.254', 'Po1': '10.168.118.241', 'Po2': '10.168.118.242', 'Po3': '10.168.118.243', 'Po4': '10.168.118.244'}
</pre>


Breaking it down using list comprehensions:
===========================================

Using Python list comprehensions allows you to use all of the things shown previously while writing it down in a very concise way. Chapter 14 from `Learning Python, 5th Edition` contains some great examples and explanations on list comprehensions.

That book also gives the following definition of list comprehensions:

'List comprehensions collect the results of applying an arbitrary expression to an iterable of values and return them in a new list.'

Here is the Python intro to list comprehensions:
https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions

There, the list comprehension is defined like this:

<pre style="font-size:12px">
A list comprehension consists of brackets containing an expression followed by a for clause, then zero or more for or if clauses. The result will be a new list resulting from evaluating the expression in the context of the for and if clauses which follow it.
</pre>

The syntax for a list comprehension is the following:

<pre style="font-size:12px">
[ expression for item in list if (not) conditional(s) ]
</pre>

Combinging list comprehensions with `join()` can be nice way to transform a string into the string you want, containing only the lines you are interested in. In the following example, we will 
```python
s = """
SoMe StRinG.   

Empty lInEs, leAding anD Trailing whiTespaces.   
  MiXed UPPER and lower case.
"""
example_list = [ line for line in s.splitlines() if line ]
```

The `example_list` now contains the following:

<pre style="font-size:12px">
['SoMe StRinG.   ', 'Empty lInEs, leAding anD Trailing whiTespaces.   ', '  MiXed UPPER and lower case.']
</pre>

The string was converted into a list. By putting in the <b>if line</b> test, we ensure that non-empty lines are not added to the list.

In case we put <b>join()</b> in front of the newly created list, we can turn it into a new string:

```python
new_string = '\n'.join([ line for line in s.splitlines() if line ] )
print(new_string)
```

The previous code will output the following:

<pre style="font-size:12px">
SoMe StRinG.   
Empty lInEs, leAding anD Trailing whiTespaces.   
  MiXed UPPER and lower case.
</pre>

In case we want to strip the output of whitespaces and turn everything to lowercase, we have to work on the expression. In this case, all we have to do is the following:

```python
new_string = '\n'.join([ line.lower().strip() for line in s.splitlines() if line ] )
print(new_string)
```

This will output the following:

<pre style="font-size:12px">
some string.
empty lines, leading and trailing whitespaces.
mixed upper and lower case.
</pre>

Now let's look at a simple example where we use a list comprehension to grab the dynamically learned MAC addresses on a switch:

```python
s = """
Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  13    0000.0c9f.f001    DYNAMIC     Po999      1       94 days, 3:54:30 ago
  13    00de.fbb9.cd41    DYNAMIC     Po992      1       94 days, 3:54:30 ago
  13    00de.fbba.aec1    DYNAMIC     Po999      1       94 days, 3:54:30 ago
  13    00de.fbba.avc1    DYNAMIC     Po996      1       94 days, 3:54:30 ago
  13    00de.fbda.a2c1    DYNAMIC     Po999      1       94 days, 3:54:30 ago
  13    00de.fb3a.avc1    DYNAMIC     Po996      1       94 days, 3:54:30 ago  
  13    7483.ef2c.bbb3    STATIC      Po1000
 870    0000.0c9f.f001    DYNAMIC     Po999      1       94 days, 3:54:30 ago
 870    00de.fbb9.cd41    DYNAMIC     Po999      1       94 days, 3:54:30 ago
"""

mac_list = [ mac.split()[1] for mac in s.splitlines()
            if 'DYNAMIC' in mac ]
```

The `mac_list` will hold the following items:
<pre style="font-size:12px">
['0000.0c9f.f001', '00de.fbb9.cd41', '00de.fbba.aec1', '00de.fbba.avc1', '00de.fbda.a2c1', '00de.fb3a.avc1', '0000.0c9f.f001', '00de.fbb9.cd41']
</pre>

If you want to narrow the MAC address list down to the MAC addresses in VLAN 13, you can simply add a condition to the list comprehension, like so:
```python
mac_list = [ mac.split()[1] for mac in s.splitlines()
            if 'DYNAMIC' in mac and
            mac.split()[0] == '13']
```

With that addition, the `mac_list` will hold the following items:

<pre style="font-size:12px">
['0000.0c9f.f001', '00de.fbb9.cd41', '00de.fbba.aec1', '00de.fbba.avc1', '00de.fbda.a2c1', '00de.fb3a.avc1']
</pre>

The configuration of some vendors can also be iterated fairly easy using list comprehensions. Take a Juniper configuration retrieved using 'show configuration | display set' for instance. The following would narrow the Juniper configuration down to lines that specify the interfaces in the routing-instances:

```python
vrf_interfaces = [ line for line in s.splitlines()
            if 'routing-instances' in line and
            'interface' in line ]
```

Similar is the case with an IOS-XR configuration that is outputted using 'show running-config formal'.

```python
ospf_interfaces = [ line for line in s.splitlines()
            if 'router ospf' in line and
            'area 0' in line ]
```


Some people love list comprehensions, some others hate it and say it makes everything look needlessly complex. 


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

The string is a snippet of an IOS-XR configuration produced using <b>show running-config formal</b> and a Juniper configuration produced using <b>show configuration | display set</b>. Let's say we want to find all the configuration lines that have anything in them related to BGP policy configuration.

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

It works, but it does not look very nice and consider the abomination when you want to look for 4, 5 or more items in a string:

```python
if 'export' in line or 'import' in line or 'route-policy' in line or 'something'  in line or 'something else' in line or 'another thing' in line:
```

Using `any()` will 'return True if any element of the iterable is true'. It allows us to specifiy the items we are interested in

```python
interesting_items = [ 'export', 'import',  ]

for line in s.splitlines():
  if any(x in line for x in interesting_items):
    print(line)
```

This produces the same output as before:

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


The `all()` method will allow for something similar. The `all()` method will 'return True if all elements of the iterable are true'. 

In the following example, we will print the line in case all the words in the list are found:

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

Obviously, we can also use `any()` and `all()` inside list comprehensions:

```python
s = """
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-import-policy in
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-export-policy out
set protocols bgp group exchange import exchange-import
set protocols bgp group exchange export exchange-export
set protocols bgp group exchange neighbor 10.0.0.1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all
"""
interesting_items = [ 'exchange', 'import',  ]

interesting_lines = [ line for line in s.splitlines() if all(x in line for x in interesting_items) ]
```

This items in `interesting_lines` would be the following:
<pre style="font-size:12px">
['set protocols bgp group exchange import exchange-import', 'set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all']
</pre>




