---
layout: post
title: JSNAPy
tags: [ python, automation, juniper, pyez, ]
image: /img/python-logo.jpg
---

https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/jsnapy/_posts/2020-8-3-jsnapy-basics.md


JSNAPy can make you feel more comfortable executing changes and verifying device state. 

It allows you to create snapshots that capture the state of your devices running Junos. After capturing the device state, you can run tests against these snapshots. These tests can tell you if there is something to worry about or not.

The thing I love using JSNAPy for the most is for pre- and post- change validation. If you are performing a change on a device that has 100+ interfaces and that is also enabled for OSPF, BGP, LDP, LLDP, LACP and much more, you spend quite some time validating that operations have returned to normal after making a change. It is error-prone, time consuming and it is no fun at all, especially if you are doing it during a nightly maintenance window.

With JSNAPy, you write out your testcases in advance. When you have your maintenance, the test cases are executed in seconds.

In this walkthrough, after covering what JSNAPy is and how it works, I will cover the most basic scenario that I think is the best way to get familiar with JSNAPy.  We will write a test case and execute that using the CLI.


## What is JSNAPy 

JSNAPy stands for Junos Snapshot Administrator in Python. It leverage the Juniper API to retrieve data from the device. You can specify an RPC or a CLI that needs to be executed. The data returned by the device is stored as a snapshot:

{:refdef: style="text-align: center;"}
![JSNAPy overview](/img/jsnapy_overview.png "JSNAPy overview")
{: refdef}


By default, the snapshots are stored on the local system. You have the option of storing captured snapshots in a database. 

The test cases that JSNAPy executes against the snapshots are written according to a fixed format (more on that later). JSNAPy can run test cases against a single snapshot or it can be made to analyze 2 snapshots.


You can use JSNAPy as a CLI tool and run the configured checks manually. This works really well when you are looking at it for the first time and still figuring out how to use it. In addition to using it on the CLI, you can use JSNAPy in your Python scripts. This makes the framework very flexible (I use it in SaltStack for example). Juniper also supplies an Ansible module for people into that.

## JSNAPy use cases

One thing that JSNAPy can be used for is to perform checks against a single snaphot from a device. This works by having JSNAPy create a snapshot and applying several tests against it. Reasons for doing this can be because you are going through an audit and you have to prove, or verify, that every device has the proper firewall filters applied. In addition to compliance checks, you can also use a single snapshot to run health checks against it. For example, are all my BGP peers up? JSNAPy can be made to capture snapshots for a single device or for groups of devices. This means that after you have invested some time into writing the checks, running these checks accross all devices can be done in minutes. Checking BGP sessions with Route-refectors, verifying that all core routers have at least 2 OSPF neihbors, verifying that certain VRFs have routes learned from the CPE etc. 


{:refdef: style="text-align: center;"}
![JSNAPy check](/img/jsnapy_health_and_audit_check.png "JSNAPy check")
{: refdef}


Another use case is using JSNAPy for pre- and post-change checks. When you are performing complex changes on multiple devices, there is usually a whole variety of things you need to make sure are working before as well as after the change. And usually, this is the case for multiple devices in your network. Using JSNAPy, you can collect state before you start your change. After collecting your initial snapshot, you can test for changes and conditions by comparing subsequent snapshots to the one you created before the change. Did I lose a BGP session anywhere? Did a BGP session bounce during my maintenance? Do I have the same amount of interfaces and LLDP neighbors listed before as well as after the change?


{:refdef: style="text-align: center;"}
![JSNAPy pre- and post-change check](/img/jsnapy_pre_post_change_check.png "JSNAPy pre- and post-change check")
{: refdef}


## Coniguring JSNAPy and writing your first test file

The only thing we'll do in this example is install JSNAPy and configure it in such a way that we can use the CLI to take snapshots and run tests.

To install JSNAPy, run <b>pip install jsnapy</b>. The <b>/etc/jsnapy/jsnapy.cfg</b> file contains the default path for configuration files, snapshots and testfiles. In my example, I am using the following configuration:

```
[DEFAULT]
config_file_path= /etc/jsnapy
snapshot_path = /var/tmp/jsnapy_snapshot
test_file_path = /var/tmp/jsnapy_test
```






## REST IS TODO STUFF

```
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

```






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
























## Links:
- https://github.com/Juniper/jsnapy