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

print(s)
```

What happens here is the following:
- netmiko is imported
- a dictionary with login information is created
- we pass the login info to the `ConnectHandler` and setup a connection to the device
- with `net_connect.send_command('show version')` we send the command to the device and store the return in the variable called 's'
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
- NAPALM is imported
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

Or in 1 line (at the cost of readability):
```python
version = str([ x.split(':')[1].strip() for x in s.splitlines() if 'Software' in x ])
```



Recommended reading:
====================

Chapter 7:
https://learning.oreilly.com/library/view/learning-python-5th/9781449355722/ch07.html#string_fundamentals