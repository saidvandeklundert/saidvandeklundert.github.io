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

JSNAPy stands for Junos Snapshot Administrator in Python. It leverage the Juniper API to retrieve data from the device. The data that is retrieved us then stored as a snapshot:

{:refdef: style="text-align: center;"}
![JSNAPy overview](/img/jsnapy_overview.png "JSNAPy overview")
{: refdef}


You write test cases according to a fixed format. JSNAPy can run these test cases against the collected snapshots.

For instance, you could create a snapshot of the configuration and run several compliance checks against it. You could, for example, check if you have the proper firewall filters applied everywhere. And instead of compliance checks, you can also use a single snapshot to run health checks against it. For instance, are all my BGP peers up?

Anything a CLI command or RPC can retrieve from a device can be stored as a snapshot.

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
























