---
layout: post
title: Getting your facts straight.
image: /img/salt_stack_logo.png
---

In Salt, grains can be a very powerful tool. One of the things you can use them for is to make your templating more effective. As a network engineer, I mostly work with proxy-minions. And even though these come with their own set of grains, I wanted to explain how you can set your own grains and why that is useful.

How to set your own grains
==========================

Setting your own grains is pretty easy. Let’s look at a basic execution module I created:
```
# more /srv/salt/_modules/example.py
def set_facts():

    proxytype = __pillar__.get("proxy", {}).get('proxytype')

    if proxytype == 'junos':
        conn = __proxy__['junos.conn']()
        software_information = conn.rpc.get_software_information()
        hostname = software_information.find('.//host-name').text
        model = software_information.find('.//product-model').text
        software_version = software_information.find('.//junos-version').text

        function = hostname[:3]
        datacenter = hostname.split('.')[-1]
    
    elif proxytype == 'napalm':
        # Collect $vendor grains here
        pass


    # building facts_dict
    facts_dict = {}
    facts_dict['hostname'] = hostname
    facts_dict['model'] = model
    facts_dict['software-version'] = software_version
    facts_dict['function'] = function
    facts_dict['datacenter'] = datacenter

    # flush old grain values
    __salt__['grains.delkey']('facts')

    # set new grain values
    __salt__['grains.setval']('facts', facts_dict)

    return facts_dict
```

Let’s break it down and cover what is happening here.

First we retrieve the proxy-minion type from the pillar. Based on this return, we branch off with the ` if proxytype == 'junos':` and use the functions required to interact with that particular proxy minion. 

We then connect to the Juniper proxy minion and execute and RPC from which we retrieve the hostname, model and software version. And because this is our own environment, we know that the hostname holds more information than just that. 

The naming scheme is such that different bits of information are put into the device name. The first three characters define the device function, so we use string slicing to obtain that one. Different data encoded into the name is separated by a ‘.’, and the last piece of information is the datacenter name. So we use `split` to retrieve that value. 

Next up is the ` elif proxytype == 'napalm':`. This was only put in there to illustrate how we could turn this into something that would work on different types of proxy minions.

We then build the `facts` dictionary and assign values to the keys. 

Using `__salt__['grains.delkey']('facts')` we flush existing values and using `__salt__['grains.setval']('facts', facts_dict)` we update the grains of the minions. After running the function, the key `facts` will return the data this function gathers. 

After putting the file in the proper directory, we use `saltutil.sync_modules`. After updating the proxy-minion, we can run the function like so:

 ```
salt rxr01.bxcs01.ams example.set_facts
…
rxr01.bxs01.ams:
    ----------
    datacenter:
        ams
    function:
        rxr
    hostname:
        rxr01.bxc01.ams
    model:
        qfx10002-72q
    software-version:
        15.1X53-D65.3
```

To look at the facts, we can issue the following command:
```
salt rxr01.bxs01.ams grains.item facts 
..
rxr01.bxs01.ams:
    ----------
    facts:
        ----------
        datacenter:
            ams
        function:
            rxr
        hostname:
            rxr01.bxs01.ams
        model:
            qfx10002-72q
        software-version:
            15.1X53-D65.3 
```

 
Using grains in templates
=========================

Using grains is templates is pretty straightforward. After storing the device function as a grain, we can assign it to a variable in a template like this:
```
{% raw %}
{%- set function = grains.facts.get('function') -%}
{% endraw %}
```

Grains can prove their use in many different ways in our templates.  Let’s look at two examples and start with an obvious one. Here, we use the grain to configure different prefix-lists based on device function:
 ```
{% raw %}
{%- if function in [ 'ebr', 'ddr', ] -%}
set policy-options prefix-list MGMT 10.1.0.0/24
{%- elif function in [ 'rxr', 'sxs', ] -%}
set policy-options prefix-list MGMT 10.2.0.0/24
{%- else -%}
set policy-options prefix-list MGMT 10.3.0.0/24
{%- endif -%}
{% endraw %}
```
By using `if function in [ 'ebr', 'ddr', ]`, I think it is both easy to read as well as extend later on.

Knowing we can access grains this way can also be useful when you are designing your pillar. For instance, now that we have the datacenter name that the node is in accessible as a grain, we can add something like this to the pillar:
```
  dc:
    ams: {pub_as: 65500, int_as: 65500, dcid: 3, cid: 40}
    par: {pub_as: 65500, int_as: 65500, dcid: 3, cid: 40}
    nyc: {pub_as: 65500, int_as: 65500, dcid: 3, cid: 40}
```

We can do a lookup and use the data in a template like this:
 ```bash
{% set dc = grains.facts.get('datacenter') -%}
{% set pub_as = pillar.dc.get(dc).get('pub_as')  -%}
{% set int_as = pillar.dc.get(dc).get('int_as')  -%}
{% set cid = pillar.dc.get(dc).get(cid')  -%}

{%- if function in [ 'ebr', 'ddr', ] -%}
set routing-options autonomous-system {{ pub_as }}
set policy-options community DC-ORIGIN members {{ pub_as }}:{{ dcid }}
{%- elif function in [ 'rxr', 'sxs', ] -%}
set routing-options autonomous-system {{ int_as }}
set policy-options community DC-ORIGIN members {{ int_as }}:{{ cid }}
{%- endif -%}
```

What we did here was retrieve the datacenter name and then use it to perform a lookup in the pillar to fetch the values that apply to the datacenter the node is placed in. After this, we use those values in configuration statements that are specific to a device type.

This is so useful, I find myself using it all the time. For syslog, radius, ntp, protocol information, anything really. 

 
Closing thoughts
================

Salt ships with a lot of tools. It enables you to store device specific information as grain data and network information as pillar data. Add Jinja with Salt extensions to this and you have an extremely powerful templating engine.

You can I have been doing it for some time now and found templating in Salt is not just very powerful, it is an absolute joy to work with as well.
