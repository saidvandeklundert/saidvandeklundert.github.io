---
layout: post
title: Arista configure session
tags: [automation, arista]
image: /img/arista_logo.png
---

To configure an <b>Arista</b>, you can jump in configuration mode and start firing off your configuration commands one after the other and have the changes take effect immediately. Quite a few people that I have spoken with, people that have been working with Arista’s for several years, took this approach without realizing that Arisa also has a better alternative to offer, especially for complex changes. 

Instead of altering the running configuration directly using <b>configure</b>, use a <b>configure session</b>. When you enter a configuration session, the changes you make are not applied immediately. This allows you to verify what the exact impact on the configuration of the device will be if you were to apply the configuration. 

Another benefit of using configuration sessions is that you can have the device commit the change with a timer upon which the configuration change is reverted unless you <b>commit</b> the configuration session a second time. 

<br>

Using a configuration session
=============================


In the next example, we will change the following access list: 

<pre style="font-size:12px">
veos01#<b>show ip access-lists</b>

IP Access List mgmt
        10 permit 10 10.0.0.0/24 host 10.0.0.1
        20 deny 20 any any log 
</pre>

We enter the configuration session and apply our change like this:

<pre style="font-size:12px">
veos01#<b>configure session acl-change</b>
veos01(config-s-acl-ch)#<b>ip access-list mgmt</b>
veos01(config-s-acl-ch-acl-mgmt)#<b>no deny 20 any any log</b>
veos01(config-s-acl-ch-acl-mgmt)#<b>permit 20 192.168.0.0/24 host 10.0.0.1</b>
veos01(config-s-acl-ch-acl-mgmt)#<b>deny 30 any any log</b>
veos01(config-s-acl-ch-acl-mgmt)#
veos01(config-s-acl-ch-acl-mgmt)#<b>exit</b>
veos01(config-s-acl-ch)#
</pre>

At this moment, we have not altered the running configuration on the device. In order to verify that our change will produce the outcome we want, we can check the diff between the configuration session and the running configuration:

<pre style="font-size:12px">
veos01(config-s-acl-ch)#<b>show session-config diffs</b>
--- system:/running-config
+++ session:/acl-change-session-config
@@ -97,7 +97,8 @@
 !
 ip access-list mgmt
    10 permit 10 10.0.0.0/24 host 10.0.0.1
<b>-   20 deny 20 any any log</b>
<b>+   20 permit 20 192.168.0.0/24 host 10.0.0.1</b>
<b>+   30 deny 30 any any log</b>
 !

veos01(config-s-acl-ch)#
</pre>

The + and – detail what configuration lines will be added and what lines will disappear. If we feel good about applying this configuration, we can use commit. If we want to be extra careful, we can add a timer to the commit, like so:

<pre style="font-size:12px">
veos01(config-s-acl-ch)#<b>commit timer 00:05:00</b>
veos01#
veos01#<b>show configuration sessions detail</b>
Maximum number of completed sessions: 0
Maximum number of pending sessions: 5

  Name                     State                    User       Terminal       PID       Commit Time Left    Description                                   
  --------------------- ------------------------ ---------- -------------- --------- ---------------------- --------------------------------------------- 
  acl-change               pendingCommitTimer                                                      4m56s                                                  

config replace of commitTimerCheckPointConfig 
</pre>

The configuration from this configuration session is now active, but it will be reverted 5 minutes after issuing the commit. The device will revert the change unless we commit the session for a second time. If we are still confident that the change did not cause any problems, we can make the change permanent using the following command:

<pre style="font-size:12px">
veos01#<b>configure session acl-change commit</b>
veos01#
veos01#show configuration sessions detail
Maximum number of completed sessions: 1
Maximum number of pending sessions: 5

  Name          State           User       Terminal       PID    Commit Time Left 
  ---------- --------------- ---------- -------------- --------- ---------------- 
  acl-change    completed                                                         

veos01#
</pre>



To stop a configuration session, or to clean up configuration sessions that others have left on the device, use the following:

<pre style="font-size:12px">
veos01#<b>show configuration sessions</b>
Maximum number of completed sessions: 1
Maximum number of pending sessions: 5

  Name                      State         User    Terminal 
  ---------------------- ------------- ---------- -------- 
  sess-1826--807408832-0    pending                        

veos01#<b>configure session sess-1826--807408832-0 abort</b>
veos01#<b>show configuration sessions</b>
Maximum number of completed sessions: 1
Maximum number of pending sessions: 5

  Name    State       User    Terminal 
  ---- ----------- ---------- -------- 

veos01#
</pre>

<br>

Using configure sessions while automating
=========================================

Another great thing about the configuration sessions is it’s us in automation. When you use configure to enter configuration mode, you are basically hoping for the best. If your script breaks or if you lose connectivity for example, only a part of your change will be completed.

With configuration sessions, there is a commit only after all the configuration was done. This way you know that if there is a commit, all the intended configuration will be committed.

Another great benefit that configure sessions has to offer is the diff. You can have your script do a test run across the devices in you network and, instead of committing the change, store the diff somewhere and abort the configuration session. This is incredibly valuable as it lets you verify if configuration remediation is required and it gives you the opportunity to see whether or not your code is working as intended against the production network.

If you are automating anything using Netmiko, use the <b>enter_config_mode</b> keyword to enter a configuration session instead of simply using configure. You can use the Python <b>uuid</b> module to ensure that the session name is unique every time you touch the device.


If you have the API enabled on the Arista, you can use NAPALM. In this case, you are using configure sessions already and you have the <b>compare_config()</b> available. The following is an example on how you could use NAPALM to verify what the effect would be on the configuration of the device without actually applying it:

{:refdef: style="text-align: center;"}
![NAPALM logo](/img/napalm_logo.png "NAPALM logo")
{: refdef}

```python
import napalm
driver = napalm.get_network_driver('eos')
device = driver(hostname='169.50.169.163', username='salt', password='salt123')
device.open()
device.load_merge_candidate(filename='/var/tmp/arista.cfg')
print(device.compare_config())
device.discard_config()
```
When we run it, we see the following output:

<pre style="font-size:12px">
(venv_said) [salt$testing testing]$ <b>python arista_example.py</b>
@@ -82,7 +82,7 @@
    no switchport
 !
 interface Loopback0
<b>-   description mgmt-interface</b>
<b>+   description arista_can_diff</b>
 !
 interface Management1
    vrf forwarding labmgmt
</pre>

If everything looks good, you can follow-up and use <b>device.commit_config()</b> to apply the configuration to the device. You can read more on NAPALM [here](www.saidvandeklundert.net/2019-09-20-napalm/). 
