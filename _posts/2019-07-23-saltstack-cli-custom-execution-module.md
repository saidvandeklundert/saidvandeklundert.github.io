---
layout: post
title: SaltStack execution modules and the CLI.
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---


In this article, I will first show you three examples on how you can use <b>SaltStack</b> to send a CLI command to a device. As we will see, different proxy minion types have their own function that you can use to send a command to a device.

After this, I’ll cover an example where we do the same thing using a single custom execution module function. We'll use the example function to send a command to devices controlled by a <b>netmiko</b>, <b>napalm</b> or <b>junos</b> proxy minion.


The Netmiko proxy minion.
=========================

Netmiko uses the execution module called `netmiko`. The function to have the proxy minion send a command to a device is called `send_command` (same as when you use Netmiko outside of Salt).

To issue a command to a device called `lab-netmiko-eos`, we do the following:

```bash
/ $ salt lab-netmiko-eos netmiko.send_command 'show version'
lab-netmiko-eos:
    Arista DCS-7050TX-64-R
    Hardware version:    01.01
    Serial number:       JPE14383000
    System MAC address:  001c.739f.9b25
    
    Software image version: 4.18.2.1F
    Architecture:           i386
    Internal build version: 4.18.2.1F-5528501.41821F
    Internal build ID:      171293b2-b2a9-40e2-a2a8-5e1fcab52f49
    
    Uptime:                 2 weeks, 3 days, 7 hours and 5 minutes
    Total memory:           3569792 kB
    Free memory:            1355716 kB
```


The NAPALM proxy minion.
========================

Sending a command to a device called `lab-napalm-eos`, which is managed using a NAPALM proxy minion, is done using `net.cli`:

```bash
/ $ salt lab-napalm-eos net.cli 'show version'
lab-napalm-eos:
    ----------
    comment:
    out:
        ----------
        show version:
            Arista DCS-7280TR-48C6-R
            Hardware version:    01.00
            Serial number:       JPE16161500
            System MAC address:  444c.a8a5.359d
            
            Software image version: 4.18.0F
            Architecture:           i386
            Internal build version: 4.18.0F-4224134.4180F
            Internal build ID:      54acf3af-762b-4cc0-83b7-3b2060a5c8c1
            
            Uptime:                 87 weeks, 6 days, 11 hours and 28 minutes
            Total memory:           7760844 kB
            Free memory:            5275120 kB
            
    result:
        True
```


As you can see, where the Netmiko proxy minion simply return a string, NAPALM returns a dictionary. 

Side note: the NAPALM proxy minion uses the API to send the command to the device. So if you want to use NAPALM to manage an Arista device for instance, you’ll have to enable the API on the device.


The Junos proxy minion.
=======================

For the Juniper proxy minion, we use `junos.cli` to issue a command to a device called `lab-junos`:

```bash
/ $ salt lab-junos junos.cli 'show version'
lab-junos:
    ----------
    message:
        
        Hostname: ar02.mx
        Model: mx240
        Junos: 16.1R3-S8
        < output omitted >
    out:
        True
```


Same as with NAPALM, the return is a dictionary and the proxy minion uses the API to send a command to the device.


The custom execution module function.
=====================================

Let's create a function in an execution module that can work with any proxy minion type. To be able to send commands to different proxy minion types, we would have to ensure that the function will:
-	Check what proxy minion type it is dealing with
-	Use the proper execution module to interface with the proxy minion
-	Handle the output from the different proxy minions

Let’s go over the following example `common.py` execution module that does just that:

```python
def cli(command):
    
    proxytype = __pillar__.get('proxy').get('proxytype')

    send_command = {
        'netmiko': 'netmiko.send_command',
        'napalm': 'net.cli',
        'junos' : 'junos.cli',
        }
    
    device_output = __salt__[send_command[proxytype]](command)

    if proxytype == 'napalm':
        return device_output['out'].get(command, None)
    elif proxytype == 'junos':        
        return device_output.get('message', None)
    elif proxytype == 'netmiko':  
        return device_output  
```


The function starts out checking the proxy minion type by looking at the pillar:

```python
    proxytype = __pillar__.get('proxy').get('proxytype')
```


Next we see a dictionary that is used as a switch statement:

```python
    send_command = {
        'netmiko': 'netmiko.send_command',
        'napalm': 'net.cli',
        'junos' : 'junos.cli',
        }
```


We use this dictionary to check what execution module we should use in order to send a command to the proxy minion.

In this function, we want to be able to send a command to a NAPALM, Netmiko or Junos proxy minion. The following makes that possible:

```python
device_output = __salt__[send_command[proxytype]](command)
```

We use `__salt__` because we will be calling another salt execution module. Since we need to be able to call the execution module that is relevant to the proxy minion we are dealing with, we do a lookup in the `send_command` dictionary. The lookup, `send_command[proxytype]`, will ensure we use to the proper function for each proxy minion type. 

The output we get in return is stored inside `device_output`. Now, the only thing left is dealing with the different return values:

```python
    if proxytype == 'napalm':
        return device_output['out'].get(command, None)
    elif proxytype == 'junos':        
        return device_output.get('message', None)
    elif proxytype == 'netmiko':  
        return device_output  
```


For the NAPALM and Juniper proxy minion, we look for proper key in the return value. In case we are dealing with the Netmiko proxy minion, we simply return the string.


Testing our custom execution module.
====================================

We name the execution module `common.py` and we place it in `/srv/salt/_modules/`. After this, we sync it to all of the proxy minions using `salt \* saltutil.sync_all`.

Now that the proxy minions have access to this new execution module, we are ready to check out the new function:

```bash
/ $ salt lab-junos common.cli 'show version'
lab-junos:
    
    Hostname: ar02.mx
    Model: mx240
    Junos: 16.1R3-S8
    < output omitted >

/ $ salt lab-napalm-eos common.cli 'show version'
lab-napalm-eos:
    Arista DCS-7280TR-48C6-R
    Hardware version:    01.00
    Serial number:       JPE16161500
    System MAC address:  444c.a8a5.359d
    
    Software image version: 4.18.0F
    Architecture:           i386
    Internal build version: 4.18.0F-4224134.4180F
    Internal build ID:      54acf3af-762b-4cc0-83b7-3b2060a5c8c1
    
    Uptime:                 87 weeks, 6 days, 11 hours and 55 minutes
    Total memory:           7760844 kB
    Free memory:            5275832 kB
    
/ $ salt lab-netmiko-eos common.cli 'show version'
lab-netmiko-eos:
    Arista DCS-7050TX-64-R
    Hardware version:    01.01
    Serial number:       JPE14383000
    System MAC address:  001c.739f.9b25
    
    Software image version: 4.18.2.1F
    Architecture:           i386
    Internal build version: 4.18.2.1F-5528501.41821F
    Internal build ID:      171293b2-b2a9-40e2-a2a8-5e1fcab52f49
    
    Uptime:                 2 weeks, 3 days, 7 hours and 34 minutes
    Total memory:           3569792 kB
    Free memory:            1398988 kB
```


Wrapping up:
============

Not only does this makes it easier for people who want to gather information from the Salt CLI, it can also simplify work in other custom execution modules. In other custom execution modules, you can call the function discussed here like this:

```python
cmd_output = __salt__['common.cli'](command)
```


You can now start working with the string inside `cmd_output` and, for instance, extract information from CLI output using regex, textfsm or do whatever else you can think of.