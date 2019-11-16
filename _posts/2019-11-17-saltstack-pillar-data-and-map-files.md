---
layout: post
title: SaltStack pillar data and map files.
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

Starting out with SaltStack, I found that the pillar interface was quite an interesting concept. The pillar is an interface designed to offer values that the master distributes to (proxy) minions. Many of the first examples I encounterd illustrated how to use this pillar interface by placing all sorts of data that is relevant to the infrastructure as a whole in the pillar for templating purposes or for use in states. And as soon as I got a few working examples of my own, I started looking for data I could put into my own pillar. 

Whenever I came across something I needed for a template, I would convert the data to JSON or YAML and expose it to all the relevant (proxy) minions as pillar data. I figured <i>why not?</i> and went ahead enriching the pillar with more and more data as use cases starting popping up.

After while though, everything in Salt got really slow. States that would take 2-10 seconds in a lab suddenly took 7 minutes in production. After a lot of troubleshooting, I figured out that all of this was caused by the size of the pillar. After putting in more than 100.000 lines of JSON, YAML and Jinja, the environment basically came to a grinding halt. The master log showed that it was constantly rendering pillar data for minions, leaving little cycles for other tasks.

The solution to this problem was pretty straightforward: putting non-sensitive data into files. In the Salt world, this is sometimes referred to as map files. This map file is a file that can contain YAML, JSON or JINJA and the data from the file can be imported into a state and/or into a template.



Working with pillar data 
========================

When I set out working with the pillar, I had several scripts that would output data to files placed inside the pillar directory. The following is a shortened example of such a file:

<pre style="font-size:12px">
/ $ more /srv/pillar/data_file.sls 
{
    "devices": {
        "device_1": "10.0.0.1",
        "device_2": "10.0.0.2",
        "device_n": "10.200.0.1"
    },
    "public_as": {
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
/ $ salt minion pillar.item public_as

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
{% set public_as = pillar['public_as'][‘ams’] -%}
{% set device_2_ip = pillar['devices’][‘ device_2’] -%}
</pre>


Working with a map file to fix my problem
=========================================

The amount of data that was being stored in the pillar grew so big that it started slowing down the entire environment. At the same time, it was being used in a lot of templates. The solution was to move the data out of the pillar and to start using map files. 

First, the scripts that generated the data were changed so that they wrote the data to a directory other than the pillar directory and the files were given the extension that signifies what the file content was. So <b>/srv/pillar/data_file.sls</b> was now moved to <b>/srv/salt/data/data_file.json</b>.

After this, all the templates were changed to import the file and lookup the data from there. It was a minor change.

Looking up the pillar data:

<pre style="font-size:12px">
{% set public_as = pillar['public_as'][‘ams’] -%}
{{ public_as }}
</pre>

Doing the same thing using a map file:

<pre style="font-size:12px">
{% import_json '/srv/salt/data/data_file.json' as data_file %}
{% set public_as = data_file['public_as'][‘ams’] -%}
{{ public_as }}
</pre>

Rendering the last example where the map file is used, gives the following output:

<pre style="font-size:12px">
/ $ salt minion slsutil.renderer default_renderer='jinja' /var/tmp/example.j2 
..
minion:   
    65001
</pre>

For <b>JSON</b>, we use <b>import_json</b> and for <b>YAML</b> we use <b>import_yaml</b>. A thing worth noting is that you can also use Jinja in the files that you are importing. 

Look at the following example <b>name-server.j2</b> file:

<pre style="font-size:12px">
{%- if "wdc" in pillar["minion_id"] -%}
{
    "name-server": {
        "ns2.example.com": {
            "datacenter": "nyc",
            "name": "ns2.example.com",
            "ipv4": "192.168.1.2"
        },
        "ns1.example.com": {
            "datacenter": "sjc",
            "name": "ns1.example.com",
            "ipv4": "192.168.1.1"
        }
    }
}
{%- endif -%}
</pre>

In the example, the Jinja will perform a pillar lookup to see if ‘wdc’ is present inside the device name. If it is, it will render the JSON that is displayed. This is a nice way to have Jinja help determine what name server is relevant to a certain datacenter. In case we want to import this file in a template, we can do something like this:

<pre style="font-size:12px">
{% import_json '/srv/salt/data/name-server.j2' as name_server %}
{%- set ns_1_example = name_server['name-server']['ns1.example.com']['ipv4'] -%}
</pre>

Closing thoughts
================

When I started out working with Salt, I mainly ran into examples that explain how to work with pillar data in templates. It is very easy, works great and gets the job done. That is, until you start to scale.

What you should keep in mind is that pillar is expensive. The master needs to render a pillar for a minion, encrypt the pillar data and the message that it uses to send pillar data to the minion. 

If you have a large environment with thousands of hosts, having a pillar that is thousands of lines can impact your performance. In those cases, you are better off storing non-sensitive data that does not need this encryption in a map file. 

Data stored as YAML or JSON and then imported into a template or state is a lot less taxing to the master. 
