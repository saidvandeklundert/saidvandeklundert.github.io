---
layout: post
title: JSNAPy
tags: [ python, automation, juniper, pyez, ]
image: /img/python-logo.jpg
---


JSNAPy can make you feel more comfortable executing changes and verifying device state. 

It allows you to create snapshots that capture the state of your devices running Junos. After capturing the device state, you can run tests against these snapshots. These tests can tell you if there is something to worry about or not.

The thing I love using JSNAPy for the most is for pre- and post- change validation. If you are performing a change on a device that has 100+ interfaces and that is also enabled for OSPF, BGP, LDP, LLDP, LACP and much more, you spend quite some time validating that operations have returned to normal after executing a change. It is error-prone, time consuming and it is no fun at all, especially if you are doing it during a nightly maintenance window.

With JSNAPy, you write out your testcases in advance. When you have your maintenance, the test cases are executed in seconds.

In this walkthrough, after covering what JSNAPy is and how it works, I will cover the most basic scenario that I think is the best way to get familiar with JSNAPy.  We will write a test case and execute that using the CLI.


## What is JSNAPy 

JSNAPy stands for Junos Snapshot Administrator in Python. It leverage the Juniper API to retrieve data from the device. You can specify an RPC or a CLI that needs to be executed. The data returned by the device is stored as a snapshot:

{:refdef: style="text-align: center;"}
![JSNAPy overview](/img/jsnapy_overview.png "JSNAPy overview")
{: refdef}


By default, the snapshots are stored on the local system. You have the option of storing captured snapshots in a database. 

The test cases that JSNAPy executes against the snapshots are written according to a fixed format (more on that later). JSNAPy can run test cases against a single snapshot or it can be made to analyze 2 snapshots.


You can use JSNAPy as a CLI tool and run the configured checks manually. Apart from working very well as a CLI tool, it is also a very convenient way to learn how JSNAPy works. In addition to using it on the CLI, you can use JSNAPy in your Python scripts. This makes the framework very flexible (I use it in SaltStack for example). Juniper also supplies an Ansible module for people into that.

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
# This file can be overwritten
# It contains default path for 
# config file, snapshots and testfiles
# If required, overwrite the path with your path
#config_file_path: path of main config file
#snapshot_path : path of snapshot file
#test_file_path: path of test file

[DEFAULT]
config_file_path= /etc/jsnapy
snapshot_path = /home/said/snapshots
test_file_path = /home/said/testfiles
```

We configure a snap test in file `/etc/jsnapy/snap_config.yaml`:

```
hosts:
  - device: 10.0.19.245
    username: lab
    passwd: test123  
tests:
  - test_bgp.yaml 
```

This test will target 1 devices and run a single test called `test_bgp.yaml`. Due to our previous configuration, the path JSNAPy expects for the test file is the following: `/home/said/testfiles/test_bgp.yaml`. 

We referenced the test in `snap_config.yaml`, but we have not created it yet. Let's start off writing a test that will inform us on whether or not a BGP peer flapped.

We instruct JSNAPy to gather the information that is returned when the `show bgp summary` command is executed and iterate every BGP peer. For every BGP peer in the return, we check the `flap-count`. If there is a difference, we trigger an error and we have JSNAPy inform us of the BGP peer that flapped:

<pre style="font-size:12px">
test_bgp_summary:

  - command: show bgp summary

  - iterate:
      xpath: bgp-peer
      id: peer-address
      tests:
        - no-diff: flap-count
          info:  "Succes! {{pre['peer-address']}} did not register flaps."
          err:  "FAIL!! {{pre['peer-address']}} flapped. Pre-change: {{pre['flap-count']}}. Post-change: {{post['flap-count']}}"
</pre>

 
 After having things setup, we run a pre and post change check:


```
/ # jsnapy --snap pre -f snap_config.yaml
Connecting to device 10.0.19.245 ................
Taking snapshot of COMMAND: show bgp summary 
/ # 
/ # jsnapy --snap post -f snap_config.yaml
Connecting to device 10.0.19.245 ................
Taking snapshot of COMMAND: show bgp summary 
/ # 
```

What happened here is JSNAPy logged into the device to make an API call. It translated the `show bgp summary` command to an RPC and stored the XML return in the snapshot directory. 

In order to run our test, we issue the following command:

```
/ # jsnapy --check pre post -f /etc/jsnapy/snap_config.yaml 
**************************** Device: 10.0.19.245 ****************************
Tests Included: test_bgp_summary 
************************* Command: show bgp summary *************************
PASS | All "flap-count" is same in pre and post snapshot [ 63 matched ]
------------------------------- Final Result!! -------------------------------
test_bgp_summary : Passed
Total No of tests passed: 1
Total No of tests failed: 0 
Overall Tests passed!!! 
```

No failures! But how do we know that our test case will detect actual BGP flaps? 

To verify that our tests work, we edit the snapshot file and change the flap-count for 2 BGP neighbors. We can do this by editing the file that is stored in the snapshot directory. In this example, it is the `/home/said/snapshots/10.0.19.245_post_show_bgp_summary.xml` file that needs to be edited. After increasing the flap-count for 2 BGP neighbors, we run the check again. This time, we get the following result:

```
/ # jsnapy --check pre post -f /etc/jsnapy/snap_config.yaml 
**************************** Device: 10.0.19.245 ****************************
Tests Included: test_bgp_summary 
************************* Command: show bgp summary *************************
FAIL!! 10.0.13.49 flapped. Pre-change: ['3']. Post-change: ['13']
FAIL!! 10.0.18.229 flapped. Pre-change: ['0']. Post-change: ['1']
FAIL | All "flap-count" is not same in pre and post snapshot [ 61 matched / 2 failed ]
------------------------------- Final Result!! -------------------------------
test_bgp_summary : Failed
Total No of tests passed: 0
Total No of tests failed: 1 
Overall Tests failed!!! 
```

Expanding the test cases can be done in multiple ways. First, let's add one additional check to BGP. We want to finish up the maintenance with exactly 0 BGP peers down. In that case, we can add an additional test in the existing `test_bgp.yaml`:

<pre style="font-size:12px">
test_bgp_summary:

  - command: show bgp summary

  - iterate:
      xpath: bgp-peer
      id: peer-address
      tests:
        - no-diff: flap-count
          info:  "Succes! {{pre['peer-address']}} did not register flaps."
          err:  "FAIL!! {{pre['peer-address']}} flapped. Pre-change: {{pre['flap-count']}}. Post-change: {{post['flap-count']}}"

  - item:
      xpath: bgp-information       
      tests:
        - is-equal: down-peer-count, 0
          info:  "Succes! None of the configured BGP peers are down."
          err:  "FAIL! There are {{post['down-peer-count']}} peers down"
</pre>

We use `item` instead of `iterate` because we are interested in the first node of the xpath. The `down-peer-count` is a sub-element `bgp-information`, so that becomes the xpath expresssion. 

We can also decide to run some additional tests on other things besides BGP. Let's create a separate file for an interface test as well as an OSPF test.

First the interface test in `/home/said/testfiles/test_interfaces.yaml`:

<pre style="font-size:12px">
test_router_interface:

  - command: show interface terse
  
  - iterate:
      xpath: physical-interface
      id: name
      tests:
        - no-diff: oper-status    
          info: "Success! The operational status of interfaces pre and post change is the same."
          err: "Fail ! Interface {{id_0}} changed from {{pre['oper-status']}} to {{post['oper-status']}}."
</pre>

Next the OSPF test in `/home/said/testfiles/test_ospf.yaml`:

<pre style="font-size:12px">
ospf_interface:
  - command: show ospf interface
  - iterate:
      xpath: ospf-interface[interface-name != "lo0.0"]
      tests:
        - is-gt: neighbor-count, 0          
          info: "Success! There is at least 1 OSPF neighbor found behind {{post['interface-name']}}"
          err: "FAIL! There are no neighbors found behind {{post['interface-name']}}"    

ospf3_interface:
  - command: show ospf3 interface
  - iterate:
      xpath: ospf3-interface[interface-name != "lo0.0"]
      tests:
        - is-gt: neighbor-count, 0          
          info: "Success! There is at least 1 OSPF neighbor found behind {{post['interface-name']}}"
          err: "FAIL! There are no neighbors found behind {{post['interface-name']}}"   
</pre>

After creating the tests, we need to plug them in `/etc/jsnapy/snap_config.yaml`. While we add the tests, let's also add another host to run the checks against:
```
hosts:
  - device: 10.253.158.251
    username: lab
    passwd: test123 
  - device: 10.45.17.7
    username: lab
    passwd: test123 
tests:
  - test_bgp.yaml 
  - test_ospf.yaml
  - test_interfaces.yaml
```

Let's run the tests:

```
/ # jsnapy --snap pre -f snap_config.yaml
Connecting to device 10.253.158.251 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show ospf interface 
Taking snapshot of COMMAND: show ospf3 interface 
Taking snapshot of COMMAND: show interface terse 
Connecting to device 10.45.17.7 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show ospf interface 
Taking snapshot of COMMAND: show ospf3 interface 
Taking snapshot of COMMAND: show interface terse 
/ # 
/ # jsnapy --snap post -f snap_config.yaml
Connecting to device 10.253.158.251 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show ospf interface 
Taking snapshot of COMMAND: show ospf3 interface 
Taking snapshot of COMMAND: show interface terse 
Connecting to device 10.45.17.7 ................
Taking snapshot of COMMAND: show bgp summary 
Taking snapshot of COMMAND: show ospf interface 
Taking snapshot of COMMAND: show ospf3 interface 
Taking snapshot of COMMAND: show interface terse 
/ # 
/ # 
/ # jsnapy --check pre post -f /etc/jsnapy/snap_config.yaml 
************************** Device: 10.253.158.251 **************************
Tests Included: test_bgp_summary 
************************* Command: show bgp summary *************************
PASS | All "flap-count" is same in pre and post snapshot [ 36 matched ]
PASS | All "down-peer-count" is equal to "0" [ 1 matched ]
************************** Device: 10.253.158.251 **************************
Tests Included: ospf_interface 
************************ Command: show ospf interface ************************
PASS | All "neighbor-count" is greater than 0" [ 8 matched ]
Tests Included: ospf3_interface 
*********************** Command: show ospf3 interface ***********************
PASS | All "neighbor-count" is greater than 0" [ 8 matched ]
************************** Device: 10.253.158.251 **************************
Tests Included: test_router_interface 
*********************** Command: show interface terse ***********************
PASS | All "oper-status" is same in pre and post snapshot [ 191 matched ]
------------------------------- Final Result!! -------------------------------
test_bgp_summary : Passed
ospf_interface : Passed
ospf3_interface : Passed
test_router_interface : Passed
Total No of tests passed: 5
Total No of tests failed: 0 
Overall Tests passed!!! 
**************************** Device: 10.45.17.7 ****************************
Tests Included: test_bgp_summary 
************************* Command: show bgp summary *************************
PASS | All "flap-count" is same in pre and post snapshot [ 49 matched ]
PASS | All "down-peer-count" is equal to "0" [ 1 matched ]
**************************** Device: 10.45.17.7 ****************************
Tests Included: ospf_interface 
************************ Command: show ospf interface ************************
PASS | All "neighbor-count" is greater than 0" [ 8 matched ]
Tests Included: ospf3_interface 
*********************** Command: show ospf3 interface ***********************
PASS | All "neighbor-count" is greater than 0" [ 8 matched ]
**************************** Device: 10.45.17.7 ****************************
Tests Included: test_router_interface 
*********************** Command: show interface terse ***********************
PASS | All "oper-status" is same in pre and post snapshot [ 73 matched ]
------------------------------- Final Result!! -------------------------------
test_bgp_summary : Passed
ospf_interface : Passed
ospf3_interface : Passed
test_router_interface : Passed
Total No of tests passed: 5
Total No of tests failed: 0 
Overall Tests passed!!! 
```
## Links:
- https://github.com/Juniper/jsnapy
https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/jsnapy/_posts/2020-8-3-jsnapy-basics.md