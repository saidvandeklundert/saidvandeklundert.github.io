---
layout: post
title: SaltStack pillar data and map files.
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

When I was starting out with SaltStack, I learned about the pillar early on. The pillar is an interface designed to offer values that the master distributes to (proxy) minions. In many of the examples floating around, the goal is to show you how to use this pillar interface. This is done by placing all sorts of data that is relevant to the infrastructure as a whole in the pillar for templating purposes or for use in states. 

The examples worked really well and I started to use the pillar interface in a lot of templates. And since it worked so well, I started looking for more and more data I could put into the pillar. Whenever I came across something I needed for a template, I would convert the data to JSON or YAML and expose it to all the relevant (proxy) minions as pillar data. I figured why not? So I went ahead and started enriching the pillar with more and more data as use cases starting popping up.

After while though, everything in Salt got really slow. States that would take 2-10 seconds in a lab suddenly took 7 minutes in production. After a lot of troubleshooting, I figured out that all of this was caused by the size of the pillar. After putting in more than 100.000 lines of JSON, YAML and Jinja, the environment basically came to a grinding halt. The master log showed that it was constantly rendering pillar data for minions, leaving little cycles for other tasks.

The solution to this problem was pretty straightforward: putting non-sensitive data into files. In the Salt world, this is sometimes referred to as map files. This map file is a file that can contain YAML, JSON or JINJA and the data from the file can be imported into a state and/or into a template.

In this article, I will first put some data in the pillar and show you how to use it in a template. After this, I will move this data to a map file.



Working with pillar data 
========================

When I set out working with the pillar, I had several scripts that would output data to files placed inside the pillar directory. The following is a shortened example of such a file:

<pre style="font-size:12px">
/ $ cat /srv/pillar/data_file.sls 
{
    "hosts": {
        "switch01": "10.0.0.1",
        "router01": "10.0.0.2",
        "server01": "10.0.0.3"
    }
}
</pre>

The file contains several hostnames and IP addresses. The file was added to the pillar using the following YAML:

<pre style="font-size:12px">
base:
  '*':
    - data_file
</pre>

With this in place, I was able to retrieve the data from the pillar like this:

<pre style="font-size:12px">
/ $ salt minion pillar.item hosts

minion:
    ----------
    hosts:
        ----------
        router01:
            10.0.0.2
        server01:
            10.0.0.3
        switch01:
            10.0.0.1
</pre>

This pillar data was accessible through states and templates using something like the following:

<pre style="font-size:12px">
{% set router01 = pillar['hosts']['router01'] -%}
{{ router01 }}
</pre>

The previous template outputs the following:

<pre style="font-size:12px">
/ $ <b>salt minion slsutil.renderer default_renderer='jinja' /var/tmp/example.j2</b>
minion:
    10.0.0.2
</pre>    


Working with a map file to fix the problem
==========================================

The amount of data that was being stored in the pillar started slowing down the entire environment. The data was being used in a lot of templates, so simply deleting it was not an option. The solution was to remove the data from the pillar and to start using map files. 

First, I moved the pillar file <b>/srv/pillar/data_file.sls</b> to <b>/srv/salt/data/data_file.json</b>. After moving the file, all the templates were changed to import the file and lookup the data from there instead of using the pillar interface. This turnd out to be a pretty minor change.

Basically, one line is added to import the map file and we change the reference to the pillar into the name we give the import. Let's change the previous example template to make it use the map file:

<pre style="font-size:12px">
{% import_json '/srv/salt/data/data_file.json' as data_file %}
{% set router01 = data_file['hosts']['router01'] -%}
{{ router01 }}
</pre>

In the first line, the map file is imported as <b>data_file</b>. After this, we lookup the value from the map file instead of performing the lookup in the pillar. So we change <b>pillar['hosts']['router01']</b> into <b>data_file['hosts']['router01']</b>.

When we render this new template, we get the following output:

<pre style="font-size:12px">
/ $ <b>salt minion slsutil.renderer default_renderer='jinja' /var/tmp/example.j2</b>
minion:
    10.0.0.2
</pre>      

When we use <b>import_json</b>, we can work with it the same way we work with a dictionary. For <b>JSON</b>, we use <b>import_json</b> and for <b>YAML</b> we use <b>import_yaml</b>.

Closing thoughts
================

When I started out working with Salt, I ran into examples that explain how to work with pillar data in templates. You can put anything in there and you will find that it is very easy, works great and gets the job done. That is, until you start to scale.

What you should keep in mind is that pillar is expensive. The master needs to render the pillar for every individual minion, encrypt the pillar data and the message that it uses to send pillar data to the minion. 

If you have a large environment with thousands of hosts, having a pillar that is very big can impact performance. In some cases, you are better off storing non-sensitive data that does not need this encryption in a map file as data that is imported into a template or state is a lot less taxing to the master when compared to pillar data. 
