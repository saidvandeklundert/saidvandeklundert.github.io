---
layout: post
title: SaltStack proxy minion troubleshooting
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

Last week, I was doing some troubleshooting in a lab when I noticed one of the proxy minions refused to go into configure mode. The approach I took when I was troubleshooting this was a very general one that works for all proxy minion types. For this reason, I thought that this small write-up can benefit everyone working with proxy minions in Salt.


The problem I had was with a netmiko proxy minion that I trying to use to apply a configuration template. The execution module I created was failing and did not offer me any clues as to what the reason was.

I opened 2 terminal sessions to the server I was running Salt on. In the first session, I killed the proxy process and then ran the following command:

<b>salt-proxy --proxyid=veos14 -l debug</b>


This will allow you to see everything the proxy minion is doing. In mother other terminal session I issued the master to run the execution method again. 

The debug output made it very clear to me what the problem was:

<pre style="font-size:12px">
[DEBUG   ] check_config_mode: u'\r\nveos14>'
[DEBUG   ] write_channel: configure sessionnetmiko-detect-drift-b633f7ca-1902-4a71-9b7f-2a79def71151
[DEBUG   ] Pattern is: veos14\-
[DEBUG   ] _read_channel_expect read_data: configure sessionnetmiko-detect-drift-b633f7ca-1902-4a71-9b7f-2a79def71151
<font color='red'>% Invalid input (privileged mode required)</font>
veos14>
[DEBUG   ] Pattern found: veos14\- configure sessionnetmiko-detect-drift-b633f7ca-1902-4a71-9b7f-2a79def71151
% Invalid input (privileged mode required)
veos14>
</pre>

The problem was that the netmiko proxy minion ssh session was not in privileged mode. The Arista in the lab was missing the <b>aaa authorization exec default local</b> configuration command. After adding this, I restarted the netmiko session and everything was working again.

Running the proxy minion with the <b>-l debug</b> option works for all proxy minion types, so you can use it for the Junos and NAPALM proxy minion as well. It has helped me figure what was going on countless of times when I was debugging execution modules, connection issues and proxy minions that failed to start.






