---
layout: post
title: Python strings and network engineering
tags: [automation, python ]
image: /img/python-logo.jpg
---

Collecting the string:
======================

There are numerous ways on how to retrieve command output from a device. Let me share 3 quick and easy ways.

1. Netmiko:

```python
from netmiko import ConnectHandler

login_info = {
    'device_type': 'cisco_nxos',
    'host':   '169.60.118.254',
    'username': 'xxx',
    'password': 'xxx',
}

net_connect = ConnectHandler(**login_info)

s = net_connect.send_command('show version')
print(s)
```

2. NAPALM:

3. Paramiko:



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



Recommended reading:
====================

Chapter 7:
https://learning.oreilly.com/library/view/learning-python-5th/9781449355722/ch07.html#string_fundamentals