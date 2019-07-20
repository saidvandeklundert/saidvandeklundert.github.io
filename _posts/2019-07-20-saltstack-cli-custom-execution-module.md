---
layout: post
title: Execution modules and the CLI.
image: /img/salt_stack_logo.jpg
---


Though automation is done best using an API and working with structured data, there could still be reasons for you to do some screen scraping. This might be because existing code depends on CLI output, because a vendor does not offer an API or for some other reason.

In this article, I will first show you examples on how you can use the SaltStack CLI to send a CLI command to a device. After this, I’ll cover an example where we use a single custom execution module function to pass a command to devices managed by a `netmiko`, `napalm` or `junos` proxy minion.

 
Passing a command to Netmiko proxy minion.
==========================================

Netmiko uses the execution module called `netmiko`. The method to have the proxy minion send a command to a device is called `send_command` (same as when you use Netmiko outside of Salt).

To issue a command to a device, we can do the following:

```bash
/ $ salt lab-netmiko-eos netmiko.send_command 'show version'
lab-netmiko-eos:
    Arista DCS-7050TX-64-R
    Hardware version:    01.01
    Serial number:       JPE14383084
    System MAC address:  001c.739f.9b25
    
    Software image version: 4.18.2.1F
    Architecture:           i386
    Internal build version: 4.18.2.1F-5529602.41821F
    Internal build ID:      171293b2-b2a9-40e2-a2a8-5e1fcab52f49
    
    Uptime:                 2 weeks, 3 days, 7 hours and 5 minutes
    Total memory:           3569792 kB
    Free memory:            1355716 kB
```


Passing a command to NAPALM proxy minion.
=========================================

Passing a command to a device managed through a NAPALM proxy minion is done `net.cli`:

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
            Serial number:       JPE16161536
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


As you can see, where the Netmiko proxy minion simply return a string, NAPALM will returns a dictionary. Also worth noting is that NAPALM uses the API to pass the command to the device. So if you want to use NAPALM to manage an Arista device for instance, you’ll have to enable the API on the device.


Passing a command to Junos proxy minion.
========================================

For the Juniper proxy minion, we use `junos.cli`:

```bash
/ $ salt lab-junos junos.cli 'show version'
lab-junos:
    ----------
    message:
        
        Hostname: dar02.ims
        Model: mx240
        Junos: 16.1R3-S8
        < output omitted >
    out:
        True
/ $
```


Same as with NAPALM, the return is a dictionary and the proxy minion uses the API to pass a command to the device.


Using our own custom execution module.
======================================

We can create an execution module that can work with any proxy minion type. This can make things easier in case we use different proxy minion types or in case we plan to migrate to another proxy minion type in the future.

To be able to pass commands to different proxy minion types, we would have to ensure that the function will:
-	Check what proxy minion type it is dealing with
-	Use the correct execution module
-	Handle the output from the different proxy minions

Let’s go over the following example `common.py` execution module that does just that:

```python
def cli(command):
    """
    Send a CLI command to a device managed via a netmiko, napalm or Junos proxy minion.    
    """
    try:
        proxytype = __pillar__.get('proxy').get('proxytype')
    except:
        return 'Could not find proxytype in pillar'

    send_command = {
        'netmiko': 'netmiko.send_command',
        'napalm': 'net.cli',
        'junos' : 'junos.cli',
        }
    
    device_output = __salt__[send_command[proxytype]](command)

    if proxytype == 'napalm':
        return device_output['out'].get(command, None)
    if proxytype == 'junos':        
        return device_output.get('message', None)
    if proxytype == 'netmiko':  
        return device_output  
```


The function starts out retrieving the proxy minion type:

```python
    try:
        proxytype = __pillar__.get('proxy').get('proxytype')
    except:
        return 'Could not find proxytype in pillar'
```


Next we see a dictionary:

```python
    send_command = {
        'netmiko': 'netmiko.send_command',
        'napalm': 'net.cli',
        'junos' : 'junos.cli',
        }
```


We can use this dictionary to check what execution module we should use to pass a command to the proxy minion in use. 

If we were to call an execution module from within a function for the individual proxy minion types, we could do it like so:

```python
__salt__['netmiko.send_command'](command)
__salt__['net.cli'](command)
__salt__['junos.cli'](command)
```


In this case, we want to be able to pass the command to NAPALM, Netmiko and Junos proxy minions. To this end, we use the following:

```python
device_output = __salt__[send_command[proxytype]](command)
```


The ` send_command[proxytype]` is a lookup in the `send_command` dictionary. This lookup will ensure we turn to the proper method for each proxy minion type. 

The output we get in return is stored inside `device_output`, so the last thing is dealing with the different return values. We saw the differences in the return values earlier, so in our function we can handle it like this:

```python
    if proxytype == 'napalm':
        return device_output['out'].get(command, None)
    if proxytype == 'junos':        
        return device_output.get('message', None)
    if proxytype == 'netmiko':  
        return device_output  
```


We put the file called `common.py` in `/srv/salt/_modules/` and we sync it to the minions using ` salt \* saltutil.sync_all`.

After ensure the proxy minions have access to this new execution module, we check out the new function:

```bash
/ $ salt lab-junos common.cli 'show version'
lab-junos:
    
    Hostname: dar02.ims
    Model: mx240
    Junos: 16.1R3-S8
    < output omitted >

/ $ salt lab-napalm-eos common.cli 'show version'
lab-napalm-eos:
    Arista DCS-7280TR-48C6-R
    Hardware version:    01.00
    Serial number:       JPE16161536
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
    Serial number:       JPE14383084
    System MAC address:  001c.739f.9b25
    
    Software image version: 4.18.2.1F
    Architecture:           i386
    Internal build version: 4.18.2.1F-5529602.41821F
    Internal build ID:      171293b2-b2a9-40e2-a2a8-5e1fcab52f49
    
    Uptime:                 2 weeks, 3 days, 7 hours and 34 minutes
    Total memory:           3569792 kB
    Free memory:            1398988 kB
```


Wrapping up:
============

It is nice to have a single way to issue commands to different proxy minions. Using a custom execution module, we can achieve just that. Not only is it easier for people that want to gather information from the Salt CLI, it allows for a smooth migration (at least in some cases) from the one proxy minion type to the other. 

Consider calling a custom execution module like this in case you are extracting information from the CLI command. If you are extracting information from CLI return output using regex or textfsm for instance, the move to another proxy minion type will spare you some trouble.

The last benefit it gave me that is worth mentioning is that if you have the Salt API opened to others, you can simply have them use this method as well. This way, moving to another proxy minion type will not break someone else’s stuff either.

