---
layout: post
title: Scrapli in SaltStack
tags: [ python, automation, scrapli, saltstack ]
image: /img/python-logo.jpg
---

Recently, I have been looking into a Python library called <b>scrapli</b>. You can use it for screen scraping and what led me to checking out this package is the fact that it is easy to tap into <b>ssh2-python</b>. This can offer quite some benefits in terms of speed and CPU utilization.


### Scrapli

Scrapli can be found <a href="https://github.com/carlmontanari/scrapli" target="_blank">here</a>. The <b>README.md</b> is pretty solid. It contains a description of what scrapli is and how you can put it to use.

In the following example, scrapli is used to send a command to an Arista device:

<pre style="font-size:12px">
from scrapli.driver.core import IOSXEDriver

my_device = {
    "host": "192.168.1.1",
    "auth_username": "xxx",
    "auth_password": "xxx",
    "auth_strict_key": False,
}

with IOSXEDriver(**my_device) as conn:
    response = conn.send_command("show version")
    print(response.result)
</pre>

With this example, you can use scrapli to log in to the device using the SSH binary on your system. You can choose several other transport mechanisms in addition to the default. In order to use change the transport to <a href="https://github.com/ParallelSSH/ssh2-python" target="_blank">ssh2-python</a>, which is the fastest option that uses SSH, the <b>transport</b> needs to be set to <b>ssh2</b>:


<pre style="font-size:12px">
from scrapli.driver.core import IOSXEDriver

my_device = {
    "host": "192.168.1.1",
    "auth_username": "xxx",
    "auth_password": "xxx",
    "auth_strict_key": False,
    "<font color='red'>transport</font>" : "<font color='red'>ssh2</font>",      
}

with IOSXEDriver(**my_device) as conn:
    response = conn.send_command("show version")
    print(response.result)
</pre>

If you are normally using Paramiko (or netmiko) for screen scraping, you will see things speed up quite a bit using this package. The following are 2 posts where the ssh2-python author compares Paramiko with his own package:<br>
<a href="https://parallel-ssh.org/post/ssh2-python/" target="_blank">The State of Python SSH Libraries</a><br>
<a href="https://parallel-ssh.org/post/parallel-ssh-libssh2/" target="_blank">parallel-ssh Clients</a><br>


### Scrapli in Salt

In SaltStack, Netmiko proxy minions use Paramiko to communicate with devices. In some drivers that NAPALM uses, you will also find that Paramiko is used for SSH sessions. 

Assuming you are using netmiko proxy minions, the following would be a quick way to utilize scrapli:

<pre style="font-size:12px">
from scrapli.driver.core import EOSDriver
from scrapli.driver.core import IOSXEDriver
from scrapli.driver.core import NXOSDriver


__virtualname__ = "scrape"


def cli(command):
    """
    Use scrapli with ssh2 as transport to send a CLI command to a device.

    Args:
        command: string that represents the command you want to send to the device

    Returns:
        string that represents the command output

    CLI example:
        salt router-1 scrape.cli "show version"    

    """
    username = __pillar__["proxy"]["username"]
    password = __pillar__["proxy"]["password"]
    device_type = __pillar__["proxy"]["device_type"]
    host = __pillar__["proxy"]["host"]

    target_device = {
        "host": host,
        "auth_username": username,
        "auth_password": password,
        "auth_strict_key": False,
        "transport" : "ssh2",        
    }

    if 'eos' in device_type:
        with EOSDriver(**target_device) as conn:
            cmd_output = conn.send_command(command).result
        
        return cmd_output
    
    elif 'ios' in device_type:     
        with IOSXEDriver(**target_device) as conn:
            cmd_output = conn.send_command(command).result
        
        return cmd_output

    elif 'nxos' in device_type:
        with NXOSDriver(**target_device) as conn:
            cmd_output = conn.send_command(command).result
        
        return cmd_output
</pre>

With a module like this synced to the proxy-minion, you will be able to call the function in other execution modules as well using something like the following:

<pre style="font-size:12px">
show_version_output = __salt__['scrape.cli']('show version')
</pre>

The gain is tremendous. On many devices performing <b>netmiko.send_command 'show version'</b> can take up to 10 seconds. When I use <b>scrape.cli 'show version'</b>, it takes between 2 and 3 seconds. The best thing though, is the impact on the CPU. Just give it a try on a server and watch the difference in CPU utilization. I was amazed.


### Final thoughts

Proxy minions memory and CPU usage can be problematic. Switching to a function that uses scrapli for screen scraping can help you deal with the CPU usage and speed things up at the same time. 

So far, I have only been using scrapli to retrieve information from Cisco and Arista devices where scraping is required and this has been working very well for me. The only caveat I see at the moment is the fact that the library that makes scrapli work really fast and efficient, ssh2-python, is no longer maintained that actively.  

Switching to using scrapli is not something that helps to address the memory requirements that come with the proxy minions. To address this, Salt is working on a delta proxy. 

Another solution to the memory issue and burden that comes with managing many proxies could be <a href="https://github.com/mirceaulinic/salt-sproxy" target="_blank">salt-sproxy</a>. Things will not happen in near parallel as you do not use the message bus, but the upside is that you have no more proxy minions to manage. I am not (yet) using sproxy, but I think scrapli is able to offer the same benefits there.