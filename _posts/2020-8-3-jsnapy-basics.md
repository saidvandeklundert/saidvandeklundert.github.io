---
layout: post
title: JSNAPy
tags: [ python, automation, juniper, pyez, ]
image: /img/python-logo.jpg
---



JSNAPy can make you feel more comfortable executing changes and verifying device state. 

It allows you to create snapshots that capture the state of your devices running Junos. After capturing the device state, you can run tests against these snapshots. These tests can tell you if there is something to worry about or not.

The thing I love using JSNAPy for the most is for pre- and post- change validation. If you are performing a change on a device that has 100+ interfaces and that is also enabled for OSPF, BGP, LDP, LLDP, LACP and much more, you spend quite some time validating that operations have returned to normal after making a change. It is error-prone, time consuming and it is no fun at all, especially if you are doing it during a nightly maintenance window.

With JSNAPy, you write out your testcases in advance. When you have your maintenance, the test cases are executed in seconds.

In this walkthrough, after covering what JSNAPy is and how it works, I will cover the most basic scenario that I think is the best way to get familiar with JSNAPy.  We will write a test case and execute that using the CLI.


## What is JSNAPy 

JSNAPy stands for Junos Snapshot Administrator in Python. It leverage the Juniper API to take snapshots of the device. Anything a CLI command or RPC can retrieve from a device can be stored as a snapshot.

You can use the snapshot and run tests against it. For instance, you could create a snapshot of the configuration and run several compliance checks against it. You could, for example, check if you have the proper firewall filters applied everywhere. And instead of compliance checks, you can also use a single snapshot to run health checks against it. For instance, are all my BGP peers up?

Another option is to write tests that analyze information from 2 snapshots. You can create 2 snapshots at different times and test if certain conditions are met. An example is where you create a snapshot before and after a change and check if all interfaces that were up before the change are still up after the change.

By default, the snapshots are stored on the local system. You can configure JSNAPy to write the snapshots to a database.


You can use JSNAPy as a CLI tool and run the cofigured checks manually. This works really well when you are looking at it for the first time and still figuring out how to use it. In addition to using it on the CLI, you can use JSNAPy in you Python scripts. This makes the framework very flexible (I use it in SaltStack for example). Juniper also supplies an Ansible module for people into that.


## Making a snapshot



<pre style="font-size:12px">
said@qfx10k-re0> show bgp summary | display xml rpc 
<rpc-reply xmlns:junos="http://xml.juniper.net/junos/15.1X53/junos">
    <rpc>
        <get-bgp-summary-information>
        </get-bgp-summary-information>
    </rpc>
    <cli>
        <banner>{master}</banner>
    </cli>
</rpc-reply>
</pre>




## How does JSNAPy work?

The jsnapy configuration file is /etc/jsnapy/jsnapy.cfg

In this example, the test cfg is in:  /var/tmp/jsnapy_test/cfg_snap.yml 



jsnapy --snap pre -f /var/tmp/cfg_snap.yml 
jsnapy --snap pre -f /var/tmp/jsnapy_test/cfg_snap.yml 
jsnapy --snap post -f /var/tmp/jsnapy_test/cfg_snap.yml 
jsnapy --check /var/tmp/jsnapy_snapshot/10.0.19.245_pre_show_bgp_summary.xml  /var/tmp/jsnapy_snapshot/10.0.19.245_post_show_bgp_summary.xml -f /var/tmp/jsnapy_test/cfg_snap.yml 



Running a snapcheck:


jsnapy --snapcheck  /var/tmp/jsnapy_snapshot/10.0.19.245_post_show_bgp_summary.xml -f /var/tmp/jsnapy_test/cfg_snap.yml 



<pre style="font-size:12px">
jsnapy --snap pre -f /var/tmp/jsnapy_test/cfg_snap.yml 
jsnapy --snap post -f /var/tmp/jsnapy_test/cfg_snap.yml 
jsnapy --check pre post -f /var/tmp/jsnapy_test/cfg_snap.yml 
</pre>


## Using JSNAPy from the CLI


<pre style="font-size:12px">
/ $ jsnapy --snap pre -f /var/tmp/jsnapy_test/cfg_snap.yml 
Connecting to device 10.0.19.245 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show interface terse 
Connecting to device 10.0.19.246 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show interface terse 
/ $ 
/ $ jsnapy --snap post -f /var/tmp/jsnapy_test/cfg_snap.yml 
Connecting to device 10.0.19.245 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show interface terse 
Connecting to device 10.0.19.246 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show interface terse 
/ $ 
/ $ jsnapy --check pre post -f /var/tmp/jsnapy_test/cfg_snap.yml 
**************************** Device: 10.0.19.245 ****************************
Tests Included: test_bgp_summary 
************************* Command: show bgp summary *************************
PASS | All "flap-count" is same in pre and post snapshot [ 63 matched ]
PASS | All "elapsed-time/@seconds" is greater than 300" [ 61 matched ]
PASS | All "down-peer-count" is same in pre and post snapshot [ 1 matched ]
**************************** Device: 10.0.19.245 ****************************
Tests Included: test_router_interface 
*********************** Command: show interface terse ***********************
PASS | All "oper-status" is same in pre and post snapshot [ 127 matched ]
PASS | All "name" in pre snapshot is present in post snapshot [ 127 matched ]
------------------------------- Final Result!! -------------------------------
test_bgp_summary : Passed
test_router_interface : Passed
Total No of tests passed: 5
Total No of tests failed: 0 
Overall Tests passed!!! 
**************************** Device: 10.0.19.246 ****************************
Tests Included: test_bgp_summary 
************************* Command: show bgp summary *************************
PASS | All "flap-count" is same in pre and post snapshot [ 63 matched ]
PASS | All "elapsed-time/@seconds" is greater than 300" [ 61 matched ]
PASS | All "down-peer-count" is same in pre and post snapshot [ 1 matched ]
**************************** Device: 10.0.19.246 ****************************
Tests Included: test_router_interface 
*********************** Command: show interface terse ***********************
PASS | All "oper-status" is same in pre and post snapshot [ 126 matched ]
PASS | All "name" in pre snapshot is present in post snapshot [ 126 matched ]
------------------------------- Final Result!! -------------------------------
test_bgp_summary : Passed
test_router_interface : Passed
Total No of tests passed: 5
Total No of tests failed: 0 
Overall Tests passed!!! 
</pre>



























Google TextFSM is a Python module that is written to make parsing text easier. It is a convenient way to turn CLI output into structured data. This blog covers a TextFSM example and shares the way in which I have been using TextFSM lately.


### TextFSM example


In this example, I will parse the output from the Arista <b>show version</b> CLI command:


Arista DCS-7050TX-64-R
Hardware version:    01.11
Serial number:       JPE16073030
System MAC address:  444c.a876.9746

Software image version: 4.20.15M
Architecture:           i386
Internal build version: 4.20.15M-13793783.42015M
Internal build ID:      f99c7df7-24c7-46f7-8bf9-ddd47ae92f4f

Uptime:                 6 weeks, 0 days, 22 hours and 4 minutes
Total memory:           3818208 kB
Free memory:            2428516 kB
</pre>

In order to extract the relevant fields from this string, I wrote a TextFSM template that I stored as a separate file. The example template I am working with in this article is stored as <b>eos_show_version.fsm</b> and contains the following:

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

The first part of the template contains the values that have to be extracted from the parsed output. In this part, every line starts with the keyword 'Value' followed by the name of the value and a regex. This regex is used to match the value we are looking for.

The second part of the template, the State definitions, is where we define the state rules. The input we feed the template is read line by line, and every line is tested against each rule we define here.

Let's look at the first value I defined:

<pre style="font-size:12px">
Value MODEL (\S*)
</pre>  

The regex matches any non-whitespace character, zero or more times. It matches the model once the line in the text has been identified.

Now a look at the first state I defined:

<pre style="font-size:12px">
  ^Arista ${MODEL}
</pre>  

This state rule will match a string that starts with 'Arista'. When the text is being parsed, the '${MODEL}' will be replaced with the regex defined under that value. The state rule essentially translates to this:

<pre style="font-size:12px">
  ^Arista (\S*)
</pre>  

On <a href="https://regex101.com/" target="_blank">regex101</a>, you can see what this expression would match:

{:refdef: style="text-align: center;"}
![regex101 textFSM](/img/regex_101_textFSM.png "regex101 textFSM")
{: refdef}

The following is a script that runs the text output through the TextFSM template:

```python
import textfsm

show_version = """
Arista DCS-7050TX-64-R
Hardware version:    01.11
Serial number:       JPE16073030
System MAC address:  444c.a876.9746

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
print(data)
```

Running this script, using texFSM version 1.1.0, returns the following:


```python
[['DCS-7050TX-64-R', '01.11', 'JPE16073030', '444c.a876.9746', '4.20.15M']]
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
from pprint import pprint

show_version = """
Arista DCS-7050TX-64-R
Hardware version:    01.11
Serial number:       JPE16073030
System MAC address:  444c.a876.9746

Software image version: 4.20.15M
Architecture:           i386
Internal build version: 4.20.15M-13793783.42015M
Internal build ID:      f99c7df7-24c7-46f7-8bf9-ddd47ae92f4f

Uptime:                 6 weeks, 0 days, 22 hours and 4 minutes
Total memory:           3818208 kB
Free memory:            2428516 kB
"""

arista_parser = Arista()
pprint(arista_parser.get_version_information(show_version))
```


This will output the following:


```python
{'hardware-version': '01.11',
 'model': 'DCS-7050TX-64-R',
 'serial': 'JPE16072020',
 'software-version': '4.20.15M',
 'system-mac': '444c.a875.9745'}
```


There are a number of advantages to doing it in this way. First of all, since the method is decoupled from collecting the string, using it in a variety of other functions or scripts can be done without too much effort.

Second is that you can use this method to parse the text regardless of how the string is collected. It could be coming from a script that uses paramiko, netmiko, a file already stored, a Salt execution module or anything really.

And lastly, it can benefit you when you are writing test cases for the various parsing methods you have. In case you are using pytest, you could write something like this:

```python
def test_get_version_information():
    out = arista_parser.get_version_information(show_version)
    assert out == {'model': 'DCS-7050TX-64-R',
                   'hardware-version': '01.11',
                   'serial': 'JPE16073030',
                   'system-mac': '444c.a876.9746',
                   'software-version': '4.20.15M'}
```

In closing, I think it is a good idea to read through the <a href="https://github.com/google/textfsm/wiki/TextFSM" target="_blank">TextFSM wiki</a>. And additionally, you should definitely check out the <a href="https://github.com/networktocode/ntc-templates/tree/master/templates" target="_blank">NTC TextFSM templates</a>. This is a great repo that you can use as inspiration. It might even already have a template for something you need. 


