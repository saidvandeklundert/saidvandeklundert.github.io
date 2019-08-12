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

The `Junos` proxy minion is leveraging <b>PyEZ</b> to communicate with the Junos API over NETCONF. Junos PyEZ is 'a microframework for Python that enables you to manage and automate devices running the Junos operating system (Junos OS)' ( from the PyEZ developer guide). 

The basic functions that come with the proxy minion are found in the execution module. In case you have ever working with PyEZ, some of the things in there should seem familiar: `https://github.com/saltstack/salt/blob/develop/salt/modules/junos.py`. 

The execution module provides you with a lot of functions that will enable you to get most of the basic things done. Sending a CLI command, issuing an RPC, changing the configuration, etc. But before diving into all that, let's start a proxy minion process first.

<br>

Setting up the proxy minion
===========================

Setting up the proxy minion requires that you create a file for the proxy minion where you specify the connection details. To check the file we use for the proxy minion in our example, we use `cat /srv/pillar/ar01_lab.sls `:

```yaml
proxy:
  username: admin
  proxytype: junos
  host: 198.18.60.41
  password: salt123
  port: 830
```  

The proxy minion needs to be referenced in the top file as well. Using `cat /srv/pillar/top.sls`, we see our example contains the following:

```yaml
base:
  ar01-lab:
    - ar01_lab
    - lab_pillar
```

Next, we start start the proxy minion as a daemon. To this end, we issue the following command:

```bash
salt-proxy -d --proxyid=ar01-lab                
```

We can now see that the proxy minion process is running:

```bash
/ $ ps ax| grep ar
20862 ?        Sl    88:13 /usr/bin/python2 /usr/bin/salt-proxy -d --proxyid=ar01-lab
```

To test and see if the Salt master can communicate with the proxy minion process, we use `salt ar01-lab test.ping`:

```yaml
ar01-lab:
    True
```    

This is not an ICMP ping. It is a test that proves the Salt master can communicate with the proxy minion. It does not prove that the proxy minion can communicate with the device. We can send a CLI command to check and see if the proxy minion process can communicate with the device using `salt ar01-lab junos.cli 'show version'` for instance.

When the proxy minion is not running or responding, consider running the proxy minion in debug mode. Running the proxy minion in debug mode will output all proxy minion actions to screen. You can see the proxy minion process starting, logging in, issuing RPCs, etc. To run the proxy minion in debug mode, use the following:

```
salt-proxy --proxyid=ar01-lab -l debug  
```

If running the proxy minion in debug mode does not offer you any clues, consider looking into the proxy minion logs and the log of the Salt master. These log files are the following:
- /var/log/salt/proxy
- /var/log/salt/master

<br>

Exploring the proxy minion
==========================


The first thing you'll probably be interested in doing is sending some commands to the device.

Let's start by issuing the `salt ar01-lab junos.cli ' show ospf neighbor interface ae2.2'` Salt CLI command:

```yaml
ar01-lab:
    ----------
    message:
        
        Address          Interface              State     ID               Pri  Dead
        50.22.118.159    ae2.2                  Full      50.22.118.241      0    30
    out:
        True
```            

Instead of a CLI command, we can also execute an RPC. Let issue the following `salt ar01-lab junos.rpc get-ospf-neighbor-information interface='ae2.2'` Salt CLI command:

```yaml
ar01-lab:
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

Not text, but structured data. By default, the Juniper proxy minion uses `jxmlease` to 'dictify' the return.


Let's try sending ICMP to another device using `salt ar01-lab junos.ping '50.22.118.15' count=5 rapid=True`:

```yaml
ar01-lab:
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

The Junos proxy minion also gathers some facts about the device by default. This facts are stored as grains data. To display the facts from the device, you can use `salt ar01-lab junos.facts`. These facts do no change that often, but you can refresh them using `salt ar01-lab junos.facts_refresh`.


<br>

Working with the configuration
==============================

As far as working with the configuration goes, the execution module has the basics covered. Let's check the template we will be working with in this example using `cat /srv/salt/templates/juniper/arp.set`:

```
set system arp passive-learning
```

This basic example template will be enough to demonstrate the use of the execution modules. For more information on how to template in Salt, check this article: [Templating for network engineers in SaltStack ](https://saidvandeklundert.net/2019-06-08-templating-for-network-engineers-in-saltstack/).

Back to our example. The configuration from the template is not (yet) present on the device we are working with. Let's render the template and load the configuration using the `junos.load` function:

```yaml
/ $  salt ar01-lab junos.load salt://templates/juniper/arp.set format='set'
ar01-lab:
    ----------
    message:
        Successfully loaded the configuration.
    out:
        True
```

The template was rendered and the configuration was _loaded_ as a candidate configuration. We can see this when we examine the device:

```
admin@ar01-lab> configure 
Entering configuration mode
The configuration has been changed but not committed

[edit]
admin@ar01-lab# show | compare 
[edit system]
+   arp {
+       passive-learning;
+   }

[edit]
```

We logged in and we performed a `show | compare`, which is an easy way of seeing what the difference is between the candidate configuration and the active configuration. But we can do the same thing using the Salt CLI:

```yaml
/ $  salt ar01-lab junos.diff 
ar01-lab:
    ----------
    message:
        
        [edit system]
        +   arp {
        +       passive-learning;
        +   }
    out:
        True
```

Before doing an actual commit, we can have the Junos management daemon (`mgd`) see if we can `commit` our configuration. Using a `commit-check`, we can have `mgd` verify the candidate configuration:

```yaml
/ $  salt ar01-lab junos.commit_check
ar01-lab:
    ----------
    message:
        Commit check succeeded.
    out:
        True
```

We see here that `mgd` has no problems with our configuration. What errors you can run into here are syntax errors, references to non-existing configuration constructs (like referencing a non-existing community in a routing-policy) and more. 

For now, let's undo what we have done so far:

```yaml        
/ $  salt ar01-lab junos.rollback
ar01-lab:
    ----------
    message:
        Rollback successful
    out:
        True        
```        

The `junos.rollback` can be used to load a previous configuration that is available on the device. When we issue a `junos.rollback`, we effectively do a `rollback 0` and we return to the current active configuration and we discard our candidate configuration. By default, any Juniper will allow us to rollback to anything ranging from 0-49.

After performing the rollback, the candidate configuration no longer exists:

```
admin@ar01-lab> configure 
Entering configuration mode

[edit]
admin@ar01-lab# quit 
Exiting configuration mode
```

From the start of the configuration section, I followed along with the `interactive-commands` log on the device. From the logs in there, we can see what exactly how the different functions interacted with the device:

```
admin@ar01-lab> monitor start interactive-commands | match NETCONF 

admin@ar01-lab> 
*** interactive-commands ***

Aug  9 20:13:55  ar01-lab mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'load-configuration action="set" format="text"'

Aug  9 20:14:00  ar01-lab mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'get-configuration compare="rollback" rollback="0" format="text"'

Aug  9 20:14:08  ar01-lab mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'commit-configuration check'

Aug  9 20:14:16  ar01-lab mgd[29316]: UI_NETCONF_CMD: User 'admin' used NETCONF client to run command 'load-configuration compare="rollback" rollback="0"'

monitor stop 

admin@ar01-lab> 
```


Instead of performing a `rollback`, we could have also used `junos.commit` to commit the candidate configuration and make our changes take effect. Let's take a different approach here and do everything in one go. The following Salt CLI command will render the template and apply it:

```yaml
/ $ salt ar01-lab junos.install_config 'salt://templates/juniper/arp.set'  mode='private' comment='salty' 
ar01-lab:
    ----------
    message:
        Successfully loaded and committed!
    out:
        True
/ $ 
```

Notice 2 optional keywords I used here. First of is the <b>mode</b>. By setting the mode to private, we ensure that the candidate configuration we create is for our current user only. This means we cannot inadvertently include configuration statements that other users have put into their candidate configuration.
Second is the <b>comment</b>. This an optional message for others to read in the commit log using `show system commit`.

After the change, we can see the following on the device:

```
admin@ar01-lab> show configuration system arp                         
passive-learning;

admin@ar01-lab> show system commit 
0   2019-08-09 20:28:42 UTC by admin via netconf
    salty
```    



Now, let's rollback:

```yaml
/ $ salt ar01-lab junos.rollback id='1'
ar01-lab:
    ----------
    message:
        Rollback successful
    out:
        True
/ $ 
```

Notice that when we used `junos.rollback`, a `rollback 1` was done <i>and</i> it was followed by a commit:

```
admin@ar01-lab> show configuration system arp    

admin@ar01-lab> show system commit               
0   2019-08-09 20:31:33 UTC by admin via netconf
1   2019-08-09 20:28:42 UTC by admin via netconf
    salty
```    

<br>

Wrapping up
===========

In this article we explored the <b>Junos</b> proxy minion and we investigated some of the execution module functions that it is provided with. 

The proxy minion works really well and allows for a seamless interaction with the Juniper API. I think it is pretty smart to tie the whole thing in with the PyEZ microframework. It gives it some maturity to the proxy minion and it will give people that have worked with PyEZ a running start.

The execution module functions are great for several reasons. First of all, they are great to play around with and will help getting to know the Junos proxy minion a little better. Additionally, you can use them for ad-hoc information gathering leveraging CLI/RPCs you already know. Add Salt's 0MQ, and you have quite a powerful tool to instantly check what is going on in your network. We did not touch all execution module function. Using `salt ar01-lab junos`, you can see what other functions exists.

But apart from applying the execution module functions immediately like we have done here, the functions can be seen as 'low-level' building blocks. When you get to writing states, the main thing you will use are the states that the Junos proxy minion comes with. Interestingly enough, these [junos state modules ](https://github.com/saltstack/salt/blob/develop/salt/states/junos.py) leverage the custom execution modules also. 

All in all, building some familiarity with the execution module will help you better understand the states that come with the minion and studying the execution modules in more detail will help you imagine how you can start writing you own custom execution modules and/or custom states to do things 100% your own way.











