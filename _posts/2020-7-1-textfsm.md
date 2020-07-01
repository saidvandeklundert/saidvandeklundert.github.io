---
layout: post
title: TextFSM
tags: [ python, automation, textfsm, ]
image: /img/python.jpg
---


Google TextFSM is a Python module that is written to make parsing text easier. It is a convenient way to turn CLI output into structured data. In this blog, I will walkthrough a TextFSM example and share the way in which I have been using TextFSM lately.


### TextFSM example


In this example, I will parse the output from the Arista <b>show version</b> CLI command:

<pre style="font-size:12px">
Arista DCS-7050TX-64-R
Hardware version:    01.11
Serial number:       JPE16072020
System MAC address:  444c.a875.9745

Software image version: 4.20.15M
Architecture:           i386
Internal build version: 4.20.15M-13793783.42015M
Internal build ID:      f99c7df7-24c7-46f7-8bf9-ddd47ae92f4f

Uptime:                 6 weeks, 0 days, 22 hours and 4 minutes
Total memory:           3818208 kB
Free memory:            2428516 kB
</pre>

In order to extract the relevant fields from this string as structured data, I wrote a TextFSM template that I stored as a separate file. The example template I am working with in this article is stored as <b>eos_show_version.fsm</b> and contains the following:

<pre style="font-size:12px">
Value MODEL (\S*)
Value HARDWARE_VERSION (\S+)
Value SERIAL (\S*)
Value SYSTEM_MAC (\S*)
Value SOFTWARE_VERSION (\S*)

Start
  ^Arista ${MODEL}
  ^Hardware\s*version:\s+${HARDWARE_VERSION}
  ^Serial\s*number:\s+${SERIAL}
  ^System\s*MAC\s*address:\s+${SYSTEM_MAC}
  ^Software\s*image version:\s+${SOFTWARE_VERSION} -> Record
</pre>

The first part of the template contains the values that have to be extracted from the parsed output. In this part, every line starts with the keyword 'Value' followed by the name of the value and a regex. This regex is used to match and extract the value we are looking for.

The second part of the template, the State definitions, is where we define the state rules. The input we feed the template is read line by line, and every line is tested against each rule we define here.

A basic script that we can use to run the text output through this TextFSM template is the following:


```python
import textfsm
from pprint import pprint

show_version = """
Arista DCS-7050TX-64-R
Hardware version:    01.11
Serial number:       JPE16072020
System MAC address:  444c.a875.9745

Software image version: 4.20.15M
Architecture:           i386
Internal build version: 4.20.15M-13793783.42015M
Internal build ID:      f99c7df7-24c7-46f7-8bf9-ddd47ae92f4f

Uptime:                 6 weeks, 0 days, 22 hours and 4 minutes
Total memory:           3818208 kB
Free memory:            2428516 kB
"""

template = open("eos_show_version.fsm")
re_table = textfsm.TextFSM(template)
data = re_table.ParseText(show_version)
pprint(data)
```

Running this script returns the following:


```python
[['DCS-7050TX-64-R', '01.11', 'JPE16072020', '444c.a875.9745', '4.20.15M']]
```

The output is a list that contains a list of the values that we are interested in. We can access the items in the list and put it in a dictionary like so:

```python
version_info = {}
version_info["model"] = data[0][0]
version_info["hardware-version"] = data[0][1]
version_info["serial"] = data[0][2]
version_info["system-mac"] = data[0][3]
version_info["software-version"] = data[0][4]
```


### Separate the parser from the rest


Something worth considering is to turn the code that parses the text into a method. You could write the method in such a way that it takes the string as input and return the desired data as output. 

Here is an example on how you could do this:


```python
import textfsm

class Arista():

    def get_version_information(self, show_version):
        with open("eos_show_version.fsm") as template:
            re_table = textfsm.TextFSM(template)
        
        data = re_table.ParseText(show_version)        

        version_info = {}
        version_info["model"] = data[0][0]
        version_info["hardware-version"] = data[0][1]
        version_info["serial"] = data[0][2]
        version_info["system-mac"] = data[0][3]
        version_info["software-version"] = data[0][4]

        return version_info
```


We can test the class using the following <b>test_arista_class.py</b>:


```python
from arista_class import Arista

show_version = """
Arista DCS-7050TX-64-R
Hardware version:    01.11
Serial number:       JPE16072020
System MAC address:  444c.a875.9745

Software image version: 4.20.15M
Architecture:           i386
Internal build version: 4.20.15M-13793783.42015M
Internal build ID:      f99c7df7-24c7-46f7-8bf9-ddd47ae92f4f

Uptime:                 6 weeks, 0 days, 22 hours and 4 minutes
Total memory:           3818208 kB
Free memory:            2428516 kB
"""

arista_parser = Arista()
print(arista_parser.get_version_information(show_version))
```


This will output the following:


```python
{'model': 'DCS-7050TX-64-R', 'hardware-version': '01.11', 'serial': 'JPE16072020', 'system-mac': '444c.a875.9745', 'software-version': '4.20.15M'}
```


There are a number of advantages to doing it in this way. First of all, since the method is decoupled from collecting the string, using it in a variety of other functions or scripts can be done without too much effort.

Second is that you can use this method to parse the text regardless of how the string is collected. It could be coming from a script that uses paramiko, netmiko, a file already stored, a Salt execution module or anything really.

And lastly, it can benefit you when you are writing test cases for the various parsing methods you have. In case your poison is pytest, you could write something like this:

```python
def test_get_version_information():
    out = arista_parser.get_version_information(show_version)
    assert out == {'model': 'DCS-7050TX-64-R',
                   'hardware-version': '01.11',
                   'serial': 'JPE16072020',
                   'system-mac': '444c.a875.9745',
                   'software-version': '4.20.15M'}
```

In closing, I think it is a good idea to read through the <a href="https://github.com/google/textfsm/wiki/TextFSM" target="_blank">TextFSM wiki</a>. And additionally, you should definitely check out the <a href="https://github.com/networktocode/ntc-templates/tree/master/templates" target="_blank">NTC TextFSM templates</a>. This is a great repo that you can use as inspiration. It might even already have a template for something you need. 


