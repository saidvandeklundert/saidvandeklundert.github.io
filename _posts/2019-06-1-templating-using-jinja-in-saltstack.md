---
layout: post
title: Templating using Jinja in Saltstack.
image: /img/salt_stack_logo.jpg
---

To me, templating in SaltStack is an absolute joy. It makes the generation of text-based configurations for networking devices very easy. This write up is to give you some tips and insights that I would have liked to have when I started templating myself. After walking you through an easy way to render templates in Salt, I will provide you with some of the basics, some practical examples and some tips. 

Iterate your template into perfection using slsutil.renderer
============================================================

As soon as you have your proxy minions set up, the first thing worth checking out is the `slsutil.renderer` utility that Salt provides you with. This will enable you to see how a template renders for a device without applying it. It offers you a quick way to iterate and try out new things in your templates:
 
```
salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2
…
proxy_minion:
    my first template
```

When calling it like this, Salt will use a local copy in `/srv/salt/templates` if it finds the file there. If there is no local copy, Salt will attempt to fetch the latest template from gitfs.

An advantage of using this tool is that you can use it to render your Jinja with access to all of the things SaltStack has to offer. This means that you can render templates that contain grains, pillar data, execution modules, etc. 


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


Another thing I use conditionals for in my templates is to have grains or pillar data determine the value a variable has. On Juniper MX and QFX, the configuration statement for LACP differs. In the below example, we fetch the model from the grains and have that decide the value of the variable:

```
{%- set model = grains.facts.get('model') -%}

{%- if 'qfx' in model -%}
    {% set ether_options_cfg = 'ether-options' %}
{%- elif 'mx' in model -%}
    {% set ether_options_cfg = 'gigether-options' %}
{%- endif -%}

set interfaces et-0/0/35 {{ ether_options_cfg }} 802.3ad ae0
```

With things setup like this, we can properly render the template against MX as well as QFX. For example, when we render the template against a QFX device, we get this:
```
salt juniper_pm slsutil.renderer salt://templates/my_first_template.j2
..
juniper_pm:
    set interfaces et-0/0/35 ether-options 802.3ad ae7
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
    shutdown
```


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
{% for nested_dict in pillar.get('nested-dict').values() -%}
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


Debugging the template
======================

The error messages you’ll run into when templates break are not always that helpful. This can be tricky with big templates or templates you have not touched in a while. 

It is nice to know that it is possible to make Salt generate a message that will end up in the proxy log. Observe the following template:

```
{%- set no_problem_here = 'just a var' -%}
{%- do salt.log.warning('debugging jinja 1: made it to line 2') -%} 
..
{%- do salt.log.warning('debugging jinja 2: In the loop that starts at line 23') -%}
..
{%- set erhm= pillar.get('lala').get('wow:wew') -%}
..
{%- do salt.log.warning('debugging jinja 3: got to the inner loop at line 32') -%}
```

Let’s render the template using `salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2':
```
..
proxy_minion:
    The minion function caused an exception: Traceback (most recent call last):
      File "/usr/lib/python2.7/site-packages/salt/minion.py", line 1660, in _thread_return
        return_data = minion_instance.executors[fname](opts, data, func, args, kwargs)
      File "/usr/lib/python2.7/site-packages/salt/executors/direct_call.py", line 12, in execute
        return func(*args, **kwargs)
      File "/usr/lib/python2.7/site-packages/salt/modules/slsutil.py", line 182, in renderer
        **kwargs
      File "/usr/lib/python2.7/site-packages/salt/template.py", line 101, in compile_template
        ret = render(input_data, saltenv, sls, **render_kwargs)
      File "/usr/lib/python2.7/site-packages/salt/renderers/jinja.py", line 70, in render
        **kws)
      File "/usr/lib/python2.7/site-packages/salt/utils/templates.py", line 169, in render_tmpl
        output = render_str(tmplstr, context, tmplpath)
      File "/usr/lib/python2.7/site-packages/salt/utils/templates.py", line 402, in render_jinja_tmpl
        buf=tmplstr)
    SaltRenderError: Jinja variable 'None' has no attribute 'get'

```
This is not really helpful. But due to the extra’s we put in, trailing the proxy log is giving us some clues to work with:

```
/ # tail -f /var/log/salt/proxy | grep 'debugging'
2019-06-01 11:09:29,010 [salt.loader.localhost.int.module.logmod                    :57  ][WARNING ][7101] debugging jinja 1: made it to line 2
2019-06-01 11:09:29,010 [salt.loader.localhost.int.module.logmod                    :57  ][WARNING ][7101] debugging jinja 2: In the loop that starts at line 23
```
We see the first two messages we put in the template, but we do not see the third one. After learning this, we simply move message number 2 and 3 closer together. This will enable us to locate where the problem is (eventually). 

The logging level is controlled via the master configuration. In this example, the setting was to start logging at the warning level. To be able to use `do salt.log.debug`, you need to set the logging level to include debugging.


Using execution modules inside templates
========================================

You are able to run custom execution modules inside templates. This will give you an enormous amount of flexibility. 

A use case that I ran into was when I was dealing with excess configuration. For instance, with some vendors, you can only delete syslog servers for which you specify the IP address that is configured: `no logging host 10.1.0.1`. But how can you template that without knowing what configuration is applied to your devices?
 
What you can do is retrieve the existing configuration from the device in order to be able to delete it:

```
{% set delete_existing = salt['em.nuke'](cmd='show running-config section ^logging', all=True) %}
{{ delete_existing }}
```
Or
```
{{ salt['em.nuke'](section='logging') }}
```

The result:

```
salt proxy_minion slsutil.renderer path='salt://templates/my_first_template.j2'
…
proxy_minion:
    no logging host 10.1.0.1
    no logging host 10.1.0.2 
```

Using the execution modules will enable you to basically do anything you can dream up in Python. Things like connecting to an external database, check operational information on the device, run a script someplace else, etc.


Using execution modules to access structured data inside a template
===================================================================

Let’s look at a Juniper proxy minion example where we issue the `get-interface-information` RPC and store that for use in the template:
```
{% set interface = ‘et-0/0/1’ %}
{% set interface_description_dict = salt['junos.rpc']('get-interface-information', interface_name = interface, descriptions = True ) %}
{% set interface_description = interface_description_dict.get('rpc_reply').get('interface-information').get('physical-interface').get('description') %}

{{ interface_description }}
 ```

Result:
```
salt proxy_minion slsutil.renderer path='salt://templates/my_first_template.j2'
…
proxy_minion:
    ----------
    interface description
```


Passing arguments into your template
====================================

Inside the templates, you can use grains data, pillar data and you can call execution modules to retrieve information from other places. 

But there is another way to provide additional data to a template. This can be provided by appending inline pillar data when you call a state and then using the template inside that state.

Example:
```
salt proxy_minion state.apply states.example pillar='{"ops": { "interface" : "et-0/0/1", "change" : "666", }}'
```

Inside the template, you can access this pillar data in the same way that you would access regular pillar data:
```
{% set interface = pillar.get('ops').get('interface') %}
```
Reasons for doing this can be because want to enable users running the state to pass it arguments or you are using the Enterprise API and want a nice and easy way to pass it data. When you are working with the API, passing the inline pillar data is just calling a state and passing it a dictionary. 

 
Import other files with context
===============================

If you have a ton of variables you are using in every template, it might be nice to know that you can dump them all in a single template and include that template elsewhere.

For instance, let’s create a template called default:
```
{%- set type = grains.facts.get('type') -%}
{%- set software_version = grains.facts.get('software-version') -%}
{%- set model = grains.facts.get('model') -%}
{%- set password_information = pillar.get('secret_data') -%}
```

We can import this in other templates like this:
```
{%- import 'templates/default.j2' as example with context -%}
{{ example.model }} 
```
Since we import the template as `example`, whenever we access a variable set in that template, we prefix it with `example.`.
The main advantages are that it keeps the templates smaller and the default file is easy to maintain in case something changes or needs to be added to multiple templates.

Additionally, another thing worth noting is that child templates have access to the variables declared in the parent template. So if we import a default template to be able to use variables everywhere, we will also be able to use those in the child templates as well.


Using Jinja in state and pillar files
=====================================

In addition to using jinja in templates, you can also use it in other places, like in your pillar and in your state files. 

The following is an example where doing something in a state file depends on whether or not ‘xyz’ is found in the device name:

```
{%- set device_name = pillar.get('device-name') -%}

{%- if 'xyz' in device_name -%}

generate_something:
  cmd.script:
    - name: /usr/bin/python
    - source: salt://utility/some_script.py
    - args: "{{ pw }}"

{%- endif %}
```


Salt has some pretty good additional extensions
===============================================

It is worth familiarizing yourself with the Jinja extensions SaltStack offers as many of them can come in quite handy. Reading about them before starting your templating efforts might help you a lot.

Some of the extensions allow you to do pretty cool things. For example, test if something is an IP address:
```
{{ '192.168.0.1' | is_ip }}
```
Using regex replace:
```
{% set my_text = 'yes, this is a TEST' %}
{{ my_text | regex_replace(' ([a-z])', '__\\1', ignorecase=True) }}
```
Raising custom errors:
```
{{ raise('Custom Error') }}
```

These are just some examples. No point in me covering all of them, just read up on the rest right here: https://docs.saltstack.com/en/latest/topics/jinja/index.html


Wrapping up
===========

These were some of the tips and examples that I really could have used when I started templating in Salt myself. I hope this has helps clarify a few things and that you have gained some insights.

If you have any additional tips, let me know via mail!
