---
layout: post
title: Python strings and network engineering
tags: [automation, python ]
image: /img/python-logo.jpg
---

This article contains several examples I could have really used after reading up on the basics in Python. After I read the first chapters of <b>Automate the Boring Stuff with Python</b> and <b>Learning Python, 5th Edition</b>, I somewhat struggled to put the concepts I read about into practice. I knew the basic Python data structures and string methods, but was not really sure on how to actually use any of this.

As a network engineer, I was intimately familiar with the CLI and had a lot of fun putting the stuff I learned into practice by working with CLI output. In this article, I will start by giving a few basic examples on how to send a command to a device and retrieve the output as a string. After this, I will give a few examples on how to work with those strings and do some basic, but interesting things with it.


Collecting the string:
======================

There are numerous ways on how to retrieve command output from a device. Let me share 2 quick and easy ways.

## Netmiko:

Netmiko is a <b>Multi-vendor library to simplify Paramiko SSH connections to network devices</b>. It makes sending a command to a device and retrieving the output very easy.

The following is an example of a very basic script that sends the 'show version' command to a device running NX-OS:

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
- we use getpass to make the script ask for a password
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

Here are some of the usual suspects I turn to most often:

  |                     | Arista EOS        | Junos         | Cisco IOS-XR     | Cisco NX-OS      | Cisco IOS
  | ------------------- | ----------------- | ------------- | ---------------- | ---------------- | --------- 
  | **device_type**     | arista_eos        | juniper       | cisco_xr         | cisco_nxos       | cisco_ios

The author of the package has his own 'getting started' here: https://pynet.twb-tech.com/blog/automation/netmiko.html

## NAPALM:

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
- we use getpass to make the script ask for a password
- the driver that details what type of device to connect to is selected
- the device object is created
- a connection to the device is opened
- the cli method is used to send a list of commands to the device (in this case, only 1 command:'show ospf neighbor')
- the connection to the device is closed
- we retrieve the output of the command from the dictionary that is returned by 'device.cli' and store it in the variable called s
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


Splitlines and split:
=====================

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

Combining split and splitlines can sometimes be enough to extract a value without even using regex. Look at the following example:

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

Outputs:

<pre style="font-size:12px">
4.20.14M
</pre>


Let's look at another example with NX OS. Have a look at the following output that is returned after issuing <b>show ipv6 ospfv3 neighbors</b> on a Cisco NX-OS:

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

Let's say we want to extract the OSPFv3 neighbor ID and interface behind which we find the neighbor. We can use the exact same approach here:

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


Isolating interesting lines and strings:
========================================

We have broken down large strings into chunks that are easy to manage. We can step trough lines and words and test if that part is worth looking into using <b>if</b>, as we did previously when we looked through the Cisco OSPF output:

<pre style="font-size:12px">
if 'FULL' in line:
</pre>

When you are working with the individual lines or words, there are certain string methods that can help you find the thing you are looking for. Let's say for instance, you want to find out if the word 'cisco' appears on a certain description. One way of checking for that is the following: 

```python
if 'cisco' in line.lower():
```

The '.lower()' method changes all the characters to lowercase:
```python
>>> 'Cisco'.lower()
'cisco'
```

This way, you will match 'Cisco' as well as 'cisco'. 

To check for lines or words that start with something, use `.startswith('xxx')`. For instance, if you want to find lines starting with `interface`:

```python
if line.startswith('xxx'):
```
Example:

```python
>>> 'interface Ethernet 4/1'.startswith('interface')
True
```

Carefull though, a gotcha might be a space at the beginning of the line:

```python
>>> ' interface Ethernet 4/1'.startswith('interface')
False
```

Using the strip method, we can remove it. In the previous example, the empty space was on the left, so we use the following:

```python
>>> ' interface Ethernet 4/1'.lstrip()
'interface Ethernet 4/1'
```

When we call `startswith()` on that, our condition tests True:

```python
>>> ' interface Ethernet 4/1'.lstrip().startswith('interface')
True
>>> 
```

```
results.splitlines()
line.replace(' ', '')
.lower()
.strip()
.startswith('interface')

line.split()[1]

string slicing

if '' in x

if any() in x

[ x for x if 'x' in x ]

lines = [line for line in lines if 'vty' in line and not 'ims' in line ]

```


Recommended reading:
====================

Chapter 7:
https://learning.oreilly.com/library/view/learning-python-5th/9781449355722/ch07.html#string_fundamentals