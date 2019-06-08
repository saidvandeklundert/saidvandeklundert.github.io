---
layout: post
title: Templating using Jinja in Saltstack.
image: /img/salt_stack_logo.jpg
---

Templating in SaltStack is an absolute joy. It makes the generation of text-based configurations for networking devices very easy. This write up is to give you some tips and insights that I could have used when I started templating myself. After walking you through an easy way to render templates in Salt, I will cover some of the basics and provide some practical examples. 


Iterate your template into perfection using slsutil.renderer
============================================================

As soon as you have your proxy minions set up, the first thing worth checking out is the `slsutil.renderer` utility that Salt provides you with. This will enable you to see how a template renders for a device:
 
```
salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2
…
proxy_minion:
    my first template
```

When calling it like this, Salt will use a local copy in `/srv/salt/templates` if it finds the file there. If there is no local copy, Salt will attempt to fetch the latest template from gitfs.

Another way I use this `slsutil.renderer` is to render the template inside an execution module before passing it to a function that can apply the configuration. Here is an example on how to use the `slsutil.renderer` inside an execution module called `common.py`:

```python
def render(template):
    template_string = __salt__['slsutil.renderer'](path=template, default_renderer='jinja')    
    return template_string        
      
```

After syncing it to the minion, you can call the function like so:
```
salt proxy_minion common.render salt://templates/my_first_template.j2
```

Most of the time I actually use the `common.render` to render templates from the CLI as well. One of the reasons is that the return of the `slsutil.renderer` does not always show all the newlines.


The basics
==========

The default Jinja delimiters are as follows:
- `{# .. #}` where you put your comments, this is not included in the output.
- `{% raw %}{%{% endraw %} .. {% raw %}%}{% endraw %}` where you put your statements, assign variables, etc. 
- `{{ .. }}` where you put expressions that will end up in the template output (print varables for instance).


Let's illustrate how we can get some of the basic things done and start with commenting, setting a variable and outputting that variable:

```
{# Setting and using a variable. #}
{% raw %}{%{% endraw %} set variable = 'string' {% raw %}%}{% endraw %}
Here we use the {{ variable }}.
```

This will render as follows:
```
arista_proxy_minion:
    
    
    Here we use the string.
```

In the next example, we retrieve grain and pillar data and output that to screen:
```
{% raw %}{%{% endraw %} set vendor = grains.get('vendor') {% raw %}%}{% endraw %}
{{ vendor }}

{% set snmp_string = pillar.get('snmp_community') %}
{{ snmp_string }}
```

After rendering the above template, this is what we get:
```
arista_proxy_minion:

    Arista
    
    s_n_m_p_!
```

Another thing worth nothing is that adding `-` to your statements allows you to deal with whitespaces. 
- `{%-` to deal with leading whitespace
- `-%}` to deal with trailing whitespace

Example on how that works out in templates:
```
Line 1.
{% set variable = 'string' %}
Line 3.
{%- set variable = 'string' %}
Line 5.
{% set variable = 'string' -%}
Line7.
{%- set variable = 'string' -%}
Line 9.
```
The above will render as follows:
```
proxy_minion:
    Line 1.
    
    Line 3.
    Line 5.
    Line7.Line 9.
```    


Conditional statements
======================

Let's assume that we have different device types in our network and that this information is stored as a grain. In the following example, through the use of conditional statements, we put all the different configurations into 1 template:

```
{%- set type = grains.facts.get('type') -%}     

{%- if type == 'cbr' -%}                        
set configuration for the cbr
{%- elif type in [ 'srr', 'xrr', ] -%}                        
set configuration for the srr or xrr
{%- else -%}                                    
set configuration for all the other roles
{%- endif -%}
```
In the preceding template, we start out retrieving a grain value. After this, we check if the grain is equal to `cbr`. If it is, that configuration is shown. If it is not, we move on to the next test where we check if the type is present in a list. If it is, that configuration is displayed. If it is not, the `else` reveals what configuration will be applied to all other device types.

The previous example shows a few basic checks, but we can also test for multiple conditions at once. This could look something like this:
```
    {% if ( software_version in allowed_versions or allowed_versions == 'all' ) and
          ( model in allowed_models or allowed_models == 'all' ) and
          ( type in allowed_types or allowed_type == 'all' ) %}
```


Another thing I use conditionals for in my templates is to have grains or pillar data determine the value a variable has. For instance, on Juniper MX and QFX, the configuration statement for LACP differs. In the below example, we fetch the model from the grains and have that decide the value of the variable:

```
{%- set model = grains.facts.get('model') -%}

{%- if 'qfx' in model -%}
    {% set ether_options_cfg = 'ether-options' %}
{%- elif 'mx' in model -%}
    {% set ether_options_cfg = 'gigether-options' %}
{%- endif -%}

set interfaces et-0/0/35 {{ ether_options_cfg }} 802.3ad ae0
```

With things setup like this, we can render the template against MX as well as QFX. For example, when we render the template against a QFX device, we get this:
```
salt juniper_pm slsutil.renderer salt://templates/my_first_template.j2
..
juniper_pm:
    set interfaces et-0/0/35 ether-options 802.3ad ae7
```


For loop
========

In the following example, we iterate a list that is generated using the `range` function:
```
{% for n in range(5) %}
interface eth{{ n }}
 description unused
 switchport access vlan 999
 shutdown
{% endfor %}
```

When we call the exection module, we can see that the template renders as follows:

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

In this example, we'll use the following pillar data:

```yaml
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

Endless possibilities here, but one of the things worth mentioning is passing a nested dictionary to a child template like so:
```
{% for nested_dict in pillar.get('nested-dict').values() -%}
{% include 'templates/some_template.j2' %}
{% endfor -%}
```

We will be able to access the nested dictionary in `some_template` like this:
```{{ nested_dict.some_key }}```

This will make it possible use the dictionary as an instruction to generate the configuration for a service. You can, for instance, attach an inline pillar when calling a state through the API and use the dictionary to pass whatever you want to the template.


Loading external files
======================

It is possible to use and load external files in a template. The following import utilities can help you load the respective file formats:
- `import_json`
- `import_text`
- `import_yaml`
 

Let’s look at the following `/var/tmp/example.yaml` file:

```yaml
Ethernet1:
  vlan: '150'
  description: 'storage'
Ethernet2:
  vlan: '150'
  description: 'storage'
Ethernet3:
  vlan: '155'
  description: 'kvm'
Ethernet4:
  vlan: '160'
  description: 'vmware'
Ethernet5:
  vlan: '200'
  description: 'staging'
```

We can load this file as a dictionary and step through it like so in our `/srv/salt/templates/example_using_yaml.j2` template:

```
{% import_yaml '/var/tmp/example.yaml' as example_yaml %}


{% for interface, d in example_yaml | dictsort %}
interface {{ interface }}
 description {{ d.description }}
 switchport access vlan {{ d.vlan }}
{% endfor %}

```

Because we used `| dictsort`, SaltStack sorted the dictionary for us and when we render this, we get the following:

```
interface Ethernet1
 description storage
 switchport access vlan 150

interface Ethernet2
 description storage
 switchport access vlan 150

interface Ethernet3
 description kvm
 switchport access vlan 155

interface Ethernet4
 description vmware
 switchport access vlan 160

interface Ethernet5
 description staging
 switchport access vlan 200
```


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

When we render this template, the device type is fetched from the grains. After this, that variable is used to determine what template to include. If we have 30 devices types, we simply make 30 files that contain the appropriate prefix-lists and ensure that the template filename contains the device type. This beats having 30 elif’s. 

And perhaps some device types do not need any additional prefix-lists. By testing the device type against a list of device types, we prevent the template from failing to try and include a file that does not exist.


Debugging the template
======================

The error messages you’ll run into when templates break are not always that helpful. This can be tricky when you are dealing with large and complex templates or templates you have not touched in a while. 

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
This can keep you busy for quite some time. But due to the extra’s we put in, trailing the proxy log will give us some clues to work with:

```
/ # tail -f /var/log/salt/proxy | grep 'debugging'
2019-06-01 11:09:29,010 [salt.loader.localhost.int.module.logmod                    :57  ][WARNING ][7101] debugging jinja 1: made it to line 2
2019-06-01 11:09:29,010 [salt.loader.localhost.int.module.logmod                    :57  ][WARNING ][7101] debugging jinja 2: In the loop that starts at line 23
```
We see the first two messages we put in the template, but we do not see the third one. After learning this, we simply move message number 2 and 3 closer together. This will enable us to locate where the problem is (eventually). 

The logging level is controlled via the master configuration. In this example, the setting was left at the default value. To be able to use `do salt.log.debug`, you need to set the logging level to include debugging. Altering the log level can be done here:
```
/ # cat /etc/salt/master | grep log_level_logfile
#log_level_logfile: warning
```

Using execution modules inside templates
========================================

You are able to run custom execution modules inside templates. This gives you an enormous amount of flexibility and there are a lot of interesting things you can do with this. 

One example is using execution modules to fetch structured data from a device when a template is rendered. Let’s look at a Juniper proxy minion example where we issue the `get-interface-information` RPC and store that for use in the template:
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

Another use case that I ran into was when I was dealing with excess configuration. For instance, with some vendors, you can only delete syslog servers for which you specify the IP address that is configured: `no logging host 10.1.0.1`. But how can you template that without knowing what configuration is applied to your devices?
 
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
I have encountered multiple reasons for wanting to attach inline pillar data. One reason was to enable users to pass arguments to a state they are running.  Another reason was when I was using the Enterprise API. I found that passing a dictionary to a state is a very easy and neat way to have an external script pass data to templates.


Import other templates with context
===================================

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

In case you just want to read up on Jinja, you can check out this site: http://jinja.pocoo.org/docs/


Wrapping up
===========

These were some of the tips and examples that I really could have used when I started templating in Salt myself. I hope this has helps clarify a few things and that you have gained some insights.

If you have any additional tips, let me know via mail. Thank you!
