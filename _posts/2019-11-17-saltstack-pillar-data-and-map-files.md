---
layout: post
title: SaltStack pillar data and map files.
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

When I was starting out with SaltStack, I found that the pillar interface was quite an interesting concept. The pillar is an interface designed to offer values that the master distributes to (proxy) minions. There are many examples to be found on the Internet. In many of the examples, the goal is to show you how to use this pillar interface. This is done by placing all sorts of data that is relevant to the infrastructure as a whole in the pillar for templating purposes or for use in states. 

The examples got me some working templates that worked really well. And since it worked so well, I started looking for more and more data I could put into the pillar. Whenever I came across something I needed for a template, I would convert the data to JSON or YAML and expose it to all the relevant (proxy) minions as pillar data. I figured <i>why not?</i> and went ahead enriching the pillar with more and more data as use cases starting popping up.

After while though, everything in Salt got really slow. States that would take 2-10 seconds in a lab suddenly took 7 minutes in production. After a lot of troubleshooting, I figured out that all of this was caused by the size of the pillar. After putting in more than 100.000 lines of JSON, YAML and Jinja, the environment basically came to a grinding halt. The master log showed that it was constantly rendering pillar data for minions, leaving little cycles for other tasks.

The solution to this problem was pretty straightforward: putting non-sensitive data into files. In the Salt world, this is sometimes referred to as map files. This map file is a file that can contain YAML, JSON or JINJA and the data from the file can be imported into a state and/or into a template.

In this article, I will first put some data in the pillar and show you how to use it in a template. After this, I will move this data to a map file.



Working with pillar data 
========================

When I set out working with the pillar, I had several scripts that would output data to files placed inside the pillar directory. The following is a shortened example of such a file:

<pre style="font-size:12px">
/ $ more /srv/pillar/data_file.sls 
{
    "as": {
        "ams": "65001",
        "par": "65002",
        "fra": "65003",
        "wdc": "65004",
        "tok": "65005"
    }
}
</pre>

In this file, there are some devices and their IP as well as several datacenters and their AS number. The file was added to the pillar using the following YAML:

<pre style="font-size:12px">
base:
  '*':
    - data_file
</pre>

With this in place, I was able to retrieve the data from any (proxy) minion like so:

<pre style="font-size:12px">
/ $ salt minion pillar.item as

minion:
    ----------
    public_as:
        ----------
        ams:
            65001
        fra:
            65003
        par:
            65002
        tok:
            65005
        wdc:
            65004
</pre>

This pillar data was accessible through states and templates using something like the following:

<pre style="font-size:12px">
{% set hostname = pillar['minion_id']  -%}
{% set dc = hostname.split('.')[1] -%}
{% set as = pillar['as'][dc] -%}
set routing-options autonomous-system {{ as }}
</pre>

The idea here is that I retrieve the hostname from the pillar and split that name into a list. Then, as the name of the datacenter is in the hostname, I use that to lookup the autonomous system number in the pillar:

<pre style="font-size:12px">
/ $ <b>salt ar01.ams slsutil.renderer default_renderer='jinja' /var/tmp/example.j2</b>
ar01.ams:
    set routing-options autonomous-system 65001
</pre>    


Working with a map file to fix the problem
==========================================

The amount of data that was being stored in the pillar started slowing down the entire environment. The data was being used in a lot of templates, so simply deleting it was not an option. The solution was to remove the data from the pillar and to start using map files. 

First, the scripts that generated the data were changed so that they wrote the data to a directory other than the pillar directory. In this process, I also gave the files the extension appropriate to it's content. So, for example, <b>/srv/pillar/data_file.sls</b> was  moved to <b>/srv/salt/data/data_file.json</b>.

After this, all the templates were changed to import the file and lookup the data from there instead of using the pillar interface. This turnd out to be a pretty minor change.

Looking up the pillar data the 'old' way:

<pre style="font-size:12px">
{% set as = pillar['as'][dc] -%}
</pre>

Doing the exact same thing using a map file:

<pre style="font-size:12px">
{% import_json '/srv/salt/data/data_file.json' as data_file %}
{% set as = data_file['as'][dc] -%}
</pre>

To change the previous template and make it use the map file, we change it to the following:

<pre style="font-size:12px">
{% import_json '/srv/salt/data/data_file.json' as data_file %}
{% set hostname = pillar['minion_id']  -%}
{% set dc = hostname.split('.')[1] -%}
{% set as = data_file['as'][dc] -%}
set routing-options autonomous-system {{ as }}
</pre>

Rendering the last example where the map file is used, gives the following output:

<pre style="font-size:12px">
/ $ <b>salt ar01.ams slsutil.renderer default_renderer='jinja' /var/tmp/example.j2</b>
ar01.ams:
    set routing-options autonomous-system 65001
</pre>   

When we use <b>import_json</b>, we can work with it the same way we work with a dictionary. For <b>JSON</b>, we use <b>import_json</b> and for <b>YAML</b> we use <b>import_yaml</b>.

Closing thoughts
================

When I started out working with Salt, I ran into examples that explain how to work with pillar data in templates. You can put anything in there and you will find that it is very easy, works great and gets the job done. That is, until you start to scale.

What you should keep in mind is that pillar is expensive. The master needs to render the pillar for every individual minion, encrypt the pillar data and the message that it uses to send pillar data to the minion. 

If you have a large environment with thousands of hosts, having a pillar that is very big can impact performance. In some cases, you are better off storing non-sensitive data that does not need this encryption in a map file as data that is imported into a template or state is a lot less taxing to the master when compared to pillar data. 
