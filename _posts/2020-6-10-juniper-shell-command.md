---
layout: post
title: Scripting Juniper shell commands in Python
tags: [ python, automation, juniper, pyez, ]
image: /img/juniper_logo.jpg
---

When you log in to a Juniper device, you generally land on the Juniper CLI. This is the command shell that most engineers are familiar with. But not all commands are available in the Juniper CLI. Sometimes you will need to interact with the CLI of the underlying OS, which is either Linux or FreeBSD. An example of a scenario that requires you to do so, and go to shell mode, is when you need to log in to a specific line card for instance. Using <b>start shell</b> drops you to shell mode. But how can you do this in a script?


### Issuing shell commands in a script

Junos PyEZ, the Python library to automate Juniper devices, also has methods available to issue shell commands. The code can be found <a href="https://github.com/Juniper/py-junos-eznc/blob/master/lib/jnpr/junos/utils/start_shell.py" target="_blank">here</a>. 

This code offers you the <b>StartShell</b> class that comes with a set of methods to log in to the Juniper device and land directly into shell mode. The SSH session that enables all this is created with Paramiko. This part of the code can be found <a href="https://github.com/Juniper/py-junos-eznc/blob/master/lib/jnpr/junos/utils/ssh_client.py" target="_blank">here</a>.

To use the methods, you create a <b>Device</b> object which you then use to create a <b>StartShell</b> object. The <b>StartShell</b> object will allow you to open up an SSH connection to the device and issue shell commands. 

To play around with the <b>StartShell</b> class, you can paste the following into a Python interctive shell (or run it as a script):

```python
from jnpr.junos.utils.start_shell import StartShell
from jnpr.junos import Device

dev=Device(host='10.175.74.101', user='salt', password='salt123')

ss = StartShell(dev)

ss.open()

ss.run('ls -ltr /var/tmp/')
print(ss.run('ls -ltr /var/tmp/')[1])
print(ss.run('cli -c "show version | no-more"')[1])
print(ss.run('logger -e testing123')[1])

ss.close()
```

The <b>ss.open</b> and <b>ss.close</b> are used to open and close the connection to the device. The <b>ss.run</b> method will send a command to the device and return a tuple that contains two items. The first item will be True in case the shell command was executed succesffully, and False otherwise. The second item contains the command output that the device returns.

The connection has a default timeout set to 30. In case you are dealing with a command that takes a long time to complete, you can pass a timeout value along with the run method. Let try this out by running a command that takes some time to complete. The log messages that are accessible via <b>show log messages</b> are stored in <b>/var/log/messages</b>. Unless they recently rolled over, chances are this file contains quite some lines. Let's look at the content of this log file using the following script:

```python
from jnpr.junos.utils.start_shell import StartShell
from jnpr.junos import Device

dev=Device(host='10.175.74.101', user='salt', password='salt123')

ss = StartShell(dev)

ss.open()

cli_1 = ss.run('cat /var/log/messages"', timeout=2)    
cli_2 = ss.run('cat /var/log/messages', timeout=90)
ss.close()


print(cli_1[0])
print(cli_2[0])

```

When we run the script, we can see the following:

```
sh-4.4# python3 ex_3.py
False
True
```

The first command did not complete because the timeout was too short. The second command, which had an increased timeout, did complete. The way this is determined currently is by checking whether or not the prompt that is returned by the SSH session matches the prompt that should be there after a command completed.

Another thing worth noting is that <b>StartShell</b> can be used with the context manager. This means you can use 'with' to open the connection. If you do, then as soon as the block of code in the 'with' statement is executed, the device connection is automatically closed: 

```python
from jnpr.junos.utils.start_shell import StartShell
from jnpr.junos import Device

dev=Device(host='10.175.74.101', user='salt', password='salt123')

ss = StartShell(dev)

with StartShell(dev) as ss:    
    log_msg = ss.run('cat /var/log/messages', timeout=90)
    log_msg_0 = ss.run('zcat /var/log/messages.0.gz', timeout=90)

print(log_msg[1])
print(log_msg_0[1])
```

Hope this helps!