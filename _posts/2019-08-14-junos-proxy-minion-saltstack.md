---
layout: post
title: Using the Junos proxy minion in SaltStack.
image: /img/salt_stack_logo.jpg
---

To enable <b>SaltStack</b> to control devices that cannot run standard salt-minion software, we turn to proxy minions. The proxy minion is a process that Salt controls as if it were a minion. In turn, the proxy minion controls a device through the API or through CLI commands.

When we want to have SaltStack control <b>Juniper</b> devices, we can choose from the following proxy minion software: `netmiko`, `napalm` or `junos`.

This article is about the <b>Junos</b> proxy minion and the focus is on using the standard functions that the minion provides you with.

<br>

Juniper proxy minion background
===============================

The `Junos` proxy minion is leveraging the <b>PyEZ</b> library to communicate with the Junos API over NETCONF. Junos PyEZ is '<i>a microframework for Python that enables you to manage and automate devices running the Junos operating system (Junos OS)</i>' ( from the PyEZ developer guide). 

The basic functions that come with the proxy minion are found in the execution module. In case you have ever working with PyEZ, some of the things in there should seem familiar: `https://github.com/saltstack/salt/blob/develop/salt/modules/junos.py`. 

The execution module provides you with a lot of functions that will enable you to get most of the basic things done. Sending a CLI, an RPC, changing the configuration, etc. Before we dive into that, let's start a proxy minion process first.

<br>

Setting up the proxy minion
===========================

Setting up the proxy minion requires that you create a file for the proxy mininon, where you specify the connection details. To check the file we use for the proxy minion in our example, we use `cat /srv/pillar/dar01_dal05_lab03.sls `:

```yaml
proxy:
  username: admin
  proxytype: junos
  host: 198.18.60.41
  password: salt123
  port: 830
minion_id: dar01-dal05-lab03
```  

The proxy minion needs to be referenced in the top file also. Using `cat /srv/pillar/top.sls`, we see it contains the following:

```yaml
base:
  dar01-dal05-lab03:
    - dar01_dal05_lab03
    - lab_pillar
```

Next, we start start the proxy minion as a daemon. To this end, we issue the following command:

```
salt-proxy -d --proxyid=dar01-dal05-lab03                
```

We can now see that the proxy minion process is running:

```bash
/ $ ps ax| grep ar
20862 ?        Sl    88:13 /usr/bin/python2 /usr/bin/salt-proxy -d --proxyid=dar01-dal05-lab03
```

To test and see if the Salt master can communicate with the proxy minion process, we use `salt dar01-dal05-lab03 test.ping`:

```yaml
dar01-dal05-lab03:
    True
```    

This is not an ICMP ping, rather, a test that proves the Salt master can communicate with the proxy minion. It does not prove that the proxy minion can communicate with the device. We can send a CLI command to check and see if the proxy minion process can communicate with the device using `salt dar01-dal05-lab03 junos.cli 'show version'` for instance.

When the proxy minion is not running or responding, consider running the proxy minion in debug mode. Running the proxy minion in debug mode will output all proxy minion actions to screen. You can see the proxy minion process starting, logging in, issuing RPCs, etc. To run the proxy minion in debug mode, use the following:

```
salt-proxy --proxyid=dar01-dal05-lab03 -l debug  
```

If running the proxy minion in debug mode does not offer you any clues, consider looking into the proxy minion logs and the log of the Salt master. These log files are the following:
- /var/log/salt/proxy
- /var/log/salt/master

<br>

Exploring the proxy minion
==========================


The first thing you'll probably be interested in doing is sending some commands to the device.

`salt dar01-dal05-lab03 junos.cli ' show ospf neighbor interface ae2.2'`
```yaml
dar01-dal05-lab03:
    ----------
    message:
        
        Address          Interface              State     ID               Pri  Dead
        50.22.118.159    ae2.2                  Full      50.22.118.241      0    30
    out:
        True
```        

`salt dar01-dal05-lab03 junos.rpc get-ospf-neighbor-information interface='ae2.2'`
```yaml
dar01-dal05-lab03:
    ----------
    out:
        True
    rpc_reply:
        ----------
        ospf-neighbor-information:
            ----------
            ospf-neighbor:
                ----------
                activity-timer:
                    30
                interface-name:
                    ae2.2
                neighbor-address:
                    50.22.118.159
                neighbor-id:
                    50.22.118.241
                neighbor-priority:
                    0
                ospf-neighbor-state:
                    Full
``` 


Sending ICMP to another device:
```yaml
/ $ salt dar01-dal05-lab03 junos.ping '50.22.118.15' count=5 rapid=True
dar01-dal05-lab03:
    ----------
    message:
        ----------
        ping-results:
            ----------
            packet-size:
                56
            ping-failure:
                no response
            probe-results-summary:
                ----------
                packet-loss:
                    100
                probes-sent:
                    5
                responses-received:
                    0
            target-host:
                50.22.118.15
            target-ip:
                50.22.118.15
    out:
        True
```        

Display the facts from the device:
`salt dar01-dal05-lab03 junos.facts`

Reload the facts dictionary from the device:
`salt dar01-dal05-lab03 junos.facts_refresh`


The following will give you all the functions available to you in the execution module that Salt provides you with:
`salt dar01-dal05-lab03 junos`

<br>

Working with the configuration
==============================

```
/ $ cat /srv/salt/templates/juniper/includes/juniper_system__arp.set
set system arp passive-learning
```

```yaml
/ $  salt dar01-dal05-lab03 junos.load salt://templates/juniper/includes/juniper_system__arp.set format='set'
dar01-dal05-lab03:
    ----------
    message:
        Successfully loaded the configuration.
    out:
        True
```

We can now see the changes on the device as well:

```
admin@dar02.ims> configure 
Entering configuration mode
The configuration has been changed but not committed

[edit]
admin@dar02.ims# show | compare 
[edit system]
+   arp {
+       passive-learning;
+   }

[edit]
```

```yaml
/ $  salt dar01-dal05-lab03 junos.diff 
dar01-dal05-lab03:
    ----------
    message:
        
        [edit system]
        +   arp {
        +       passive-learning;
        +   }
    out:
        True
```


```yaml
/ $  salt dar01-dal05-lab03 junos.commit_check
dar01-dal05-lab03:
    ----------
    message:
        Commit check succeeded.
    out:
        True
```


```yaml        
/ $  salt dar01-dal05-lab03 junos.rollback
dar01-dal05-lab03:
    ----------
    message:
        Rollback successful
    out:
        True        
```        

After the rollback, the configuration is no longer changed:

```
admin@dar02.ims> configure 
Entering configuration mode

[edit]
admin@dar02.ims# quit 
Exiting configuration mode
```

If we would have been following along with the interactive-commands log on the device:
```
admin@dar02.ims> monitor start interactive-commands | match NETCONF 

admin@dar02.ims> 
*** interactive-commands ***

Aug  9 20:13:55  dar02.ims mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'load-configuration action="set" format="text"'

Aug  9 20:14:00  dar02.ims mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'get-configuration compare="rollback" rollback="0" format="text"'

Aug  9 20:14:08  dar02.ims mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'commit-configuration check'

Aug  9 20:14:16  dar02.ims mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'load-configuration compare="rollback" rollback="0"'

monitor stop 

admin@dar02.ims> 
```




Alternatively:

```yaml
/ $ salt dar01-dal05-lab03 junos.install_config 'salt://templates/juniper/includes/juniper_system__arp.set'  mode='private' comment='yolo' 
dar01-dal05-lab03:
    ----------
    message:
        Successfully loaded and committed!
    out:
        True
/ $ 
```

After the change:
```
admin@dar02.ims> show configuration system arp  

admin@dar02.ims> 

admin@dar02.ims> show configuration system arp                         
passive-learning;

admin@dar02.ims> show system commit 
0   2019-08-09 20:28:42 UTC by admin via netconf
    yolo
```    



Now, let's rollback:

```yaml
/ $ salt dar01-dal05-lab03 junos.rollback id='1'
dar01-dal05-lab03:
    ----------
    message:
        Rollback successful
    out:
        True
/ $ 
```

Rollback 1 was done and followed by a commit:

```
admin@dar02.ims> show configuration system arp    

admin@dar02.ims> show system commit               
0   2019-08-09 20:31:33 UTC by admin via netconf
1   2019-08-09 20:28:42 UTC by admin via netconf
    yolo
```    



( Listed at the top of this module are the two main dependencies: junos-eznc, jxmlease )