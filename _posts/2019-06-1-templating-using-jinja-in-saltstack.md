---
layout: post
title: Templating using Jinja in Saltstack.
image: /img/salt_stack_logo.jpg
---

To me, templating in SaltStack is an absolute joy. It makes the generation of text-based configurations for networking devices very easy. This write up is to give you some tips and insights that I would have liked to have when I started templating myself. After walking you through an easy way to render templates in Salt, I will provide you with some practical examples and tips. 

Quickly iterate your template using slsutil.renderer
====================================================

As soon as you have your proxy minions setup, the first thing worth checking out is the `slsutil.renderer` utility that Salt provides you with. This will enable you to see how a template renders for a device (without actually applying it). It offers you a quick way to iterate and try out new things in your templates:
 
```
salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2
…
proxy_minion:
    my first template
```

When calling it like this, Salt will use a local copy in `/srv/salt/templates` if it finds the file there. If there is no local copy, Salt will attempt to fetch the latest template from gitfs.




Some basics
===========

Some basics to get you started include setting variables and retrieving grains as well as pillar data:

```
{# Setting a variable. #}
{%- set variable = 'string' -%}

{# Using that variable. #}
Here we use the {{ variable }}.

{# Performing Python string method on that variable: #}
{{ variable | upper }}

{# Using grains data. #}
{%- set vendor = grains.get('vendor') -%}
{{ vendor }}

{# Get  and use data from the pillar. #}
{%- set snmp_string = pillar.get('snmp_string') -%}
{{ snmp_string }}

{# Using an IP stored in the pillar. #}
{{ pillar.get('server').get('primary').get('primary_ip') }}

{# grabbing/using pillar data and extracting the IP #}
{%- set server_ip = pillar.get('server').get('primary').get('primary_ip').split('/')[0] -%}
{{ server_ip }}

```

After rendering the above template, this is what we get:
```
/ # salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2
proxy_minion:
    Here we use the string.
    
    STRING
    Arista
    snmp_string
    
    10.2.2.1/24
    10.2.2.1
```


Setting variables based on grains or pillar data:
=================================================

Using grains or pillar data to control the value of a variable can be done like this:

```
{%- set model = grains.facts.get('model') -%}

{%- if 'qfx' in model -%}
    {% set ether_options_cfg = 'ether-options' %}
{%- elif 'mx' in model -%}
    {% set ether_options_cfg = 'gigether-options' %}
{%- endif -%}

set interfaces et-0/0/35 {{ ether_options_cfg }} 802.3ad ae0
```

On Juniper MX and QFX, the configuration statement for LACP differs. In the above example, we fetch the model from the grains and have that decide the value of the variable. 

With things setup like this, we can properly render the template against MX as well as QFX. For example, when we render the template against a QFX device, we get this:
```
salt juniper_pm slsutil.renderer salt://templates/my_first_template.j2
..
juniper_pm:
    set interfaces et-0/0/35 ether-options 802.3ad ae7
```


Conditional statements
======================

Let's assume that we have different device types in our network and that we have stored that role as a grain. Through the use of conditional statements, we are able to fit them all into 1 template:

```
{# retrieve grain and store that value in type #}
{%- set type = grains.facts.get('type') -%}     

{# test if the type equals cbr #}
{%- if type == 'cbr' -%}                        

set configuration for the cbr

{# test if type is found in the list [ 'srr', 'xrr', ] #}
{%- elif type in [ 'srr', 'xrr', ] -%}                        

set configuration for the srr and xrr

{# use the following as a default for other types #}
{%- else -%}                                    

set configuration for all the other roles

{%- endif -%}
```

We can also test multiple conditions at once. In Jinja, this could look something like this:
```
    {% if ( software_version in allowed_versions or allowed_versions == 'all' ) and
          ( model in allowed_models or allowed_models == 'all' ) and
          ( type in allowed_types or allowed_type == 'all' ) %}
```


For loop
========

A simple for loop:
```
{% for n in range(5) -%}
interface eth{{ n }}
 description unused
 switchport access vlan 999
 shutdown
{% endfor %}
```

When rendering from the CLI, for some reasons the newlines disappear. Issuing a ` salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2` will show the following:
```
proxy_minion:
    interface eth0 description unused switchport access vlan 999 shutdown interface eth1 description unused switchport access vlan 999 shutdown interface eth2 description unused switchport access vlan 999 shutdown interface eth3 description unused switchport access vlan 999 shutdown interface eth4 description unused switchport access vlan 999 shutdown
```

This changes when it is rendered inside an execution module. Example on how to use the `slsutil.renderer` inside an execution module called 'common.py':
```
def render(template):
    template_string = __salt__['slsutil.renderer'](path=template, default_renderer='jinja')    
    return template_string        
      
```

Now it renders like this:

```
 # salt proxy_minion common.render salt://templates/my_first_template.j2
[DEBUG   ] LazyLoaded nested.output
proxy_minion:
    interface eth0
    description unused
    switchport access vlan 999
    shutdown
    interface eth1
    description unused
    switchport access vlan 999
    shutdown
    interface eth2
    description unused
    switchport access vlan 999
    shutdown
    interface eth3
    description unused
    switchport access vlan 999
    shutdown
    interface eth4
    description unused
    switchport access vlan 999
    shutdown```


Stepping through a dictionary
=============================


Imagine the following is added to your pillar:

```
some_dict:    
  key_1:
    'value_a'  
  key_2:
    'value_b'
  key_3:
    'value_c'
```

We can step through this dictionary like so:
```
{%- for key, value in pillar.get('some_dict').items() -%} 
{{ key }} {{ value }}
{% endfor %}  
```
You can also fill in a template while you are stepping through a dictionary. In the following example, we step through a dictionary and pass it nested dictionary to the template:
```
{% for nested_dict in pillar.get('direct-link').values() -%}
{% include 'templates/some_template.j2' %}
{% endfor -%}
```

We will be able to access the dictionary in `some_template` like so:
```{{ nested_dict.some_key }}```


Using grains or pillar data to include other files into the template
====================================================================

To keep things manageable, you will eventually start working with child templates. But it can happen that after a while, even the child templates grow into thousands of lines. 

Let’s suppose that a prefix list differs per device type. The following is an easy way to create a separate prefix-list template for every device while inheriting them in the same ‘master’ template:
```
{%- set type = grains.facts.get('type') -%}
{%- if type in [ 'brr', 'adr', 'xrr', ] %}
{% include 'templates/'~type~'_prefix_list.set' %}
{%- endif -%}
```

When we run this, the device type is fetched from the grains. After this, that variable is used to determine what template to include:
```
salt proxy_minion slsutil.renderer path='salt://templates/my_first_template.j2'
..
proxy_minion:
    set policy-options prefix-list MGMT 10.0.0.1/32
```
If we have 30 devices types, we simply make 30 files that contain the appropriate prefix-lists and ensure that the template filename contains the device type. This beats having 30 elif’s. 

And perhaps some device types do not need any additional prefix-lists. By testing the device type against a list of device types, we prevent the template from failing to try and include a file that does not exist.

