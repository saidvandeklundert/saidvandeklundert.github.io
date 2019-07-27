---
layout: post
title: Templating for network engineers in SaltStack.
image: /img/salt_stack_logo.jpg
---

Templating in SaltStack is an absolute joy. It makes the generation of text-based configurations for networking devices very easy. This write up is to give you some tips and insights that I could have used when I started templating myself. After walking you through an easy way to render templates in Salt, I will cover some of the basics and provide some practical examples. 

[Iterate your template into perfection using slsutil.renderer](#iterate-your-template-into-perfection-using-slsutilrenderer)<br>
[The basics](#the-basics)<br>
[The if statement and testing strings for conditions](#the-if-statement)<br>
[Conditional statements](#conditional-statements)<br>
[For loop](#for-loop)<br>
[String slicing](#string-slicing)<br>
[Splitting a string](#splitting-a-string)<br>
[Stepping through a dictionary](#stepping-through-a-dictionary)<br>
[Loading external files](#loading-external-files)<br>
[Using grains to perform a lookup in the pillar](#using-grains-to-perform-a-lookup-in-the-pillar)<br>
[Using grains or pillar data to include other files into the template](#using-grains-or-pillar-data-to-include-other-files-into-the-template)<br>
[Debugging the template](#debugging-the-template)<br>
[Using execution modules inside templates](#using-execution-modules-inside-templates)<br>
[Passing arguments into your template](#passing-arguments-into-your-template)<br>
[Import other templates with context](#import-other-templates-with-context)<br>
[Using Jinja in state and pillar files](#using-jinja-in-state-and-pillar-files)<br>
[Salt has some pretty good additional extensions](#salt-has-some-pretty-good-additional-extensions)<br>
[Wrapping up](#wrapping-up)<br>

Iterate your template into perfection using slsutil.renderer
============================================================


As soon as you have your proxy minions set up, the first thing worth checking out is the `slsutil.renderer` utility that Salt provides you with. This will enable you to see how a template renders for a device:
 
```
salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2 default_renderer='jinja'
…
proxy_minion:
    my first template
```

When calling it like this, Salt will use a local copy in `/srv/salt/templates` if it finds the file there. If there is no local copy, Salt will attempt to fetch the latest template from gitfs.

Another way I use this `slsutil.renderer` is to render the template inside an execution module before passing it to a function that can apply the configuration. Here is an example on how to use the `slsutil.renderer` inside an execution module called `common.py`:

```python
def render(template):
    template_string = __salt__['slsutil.renderer'](path=template, default_renderer='jinja')  
    # do Pythonic things here  
    return template_string              
```

After syncing it to the minion, you can call the function like so:
```
salt proxy_minion common.render salt://templates/my_first_template.j2
```


The basics
==========


The default Jinja delimiters are as follows:
- `{# .. #}` where you put your comments, this is not included in the output.
- `{% raw %}{%{% endraw %} .. {% raw %}%}{% endraw %}` where you put your statements, assign variables, etc. 
- `{% raw %}{{{% endraw %} .. {% raw %}}}{% endraw %}` where you put expressions that will end up in the template output (print varables for instance).


Let's illustrate how we can get some of the basic things done and start with commenting, setting a variable and outputting that variable:

```
{% raw %}{# Setting and using a variable. #}
{% set variable = 'string' %}
Here we use the {{ variable }}.{% endraw %}
```

This will render as follows:

```yaml
arista_proxy_minion:
    
    Here we use the string.
```

In the next example, we retrieve grain and pillar data and output that to screen:
```
{% raw %}{% set vendor = grains.get('vendor') %}
{{ vendor }}

{% set snmp_string = pillar.get('snmp_community') %}
{{ snmp_string }}{% endraw %}
```

After rendering the above template, this is what we get:

```yaml
arista_proxy_minion:

    Arista
    
    s_n_m_p_!
```

Another thing worth noting is that adding `-` to your statements allows you to deal with whitespaces. 
- `{% raw %}{%-{% endraw %}` to deal with leading whitespace
- `{% raw %}-%}{% endraw %}` to deal with trailing whitespace

Example on how that works out in templates:

```
Line 1.
{% raw %}{% set variable = 'string' %}
Line 3.
{%- set variable = 'string' %}
Line 5.
{% set variable = 'string' -%}
Line7.
{%- set variable = 'string' -%}{% endraw %}
Line 9.
```

The above will render as follows:

```yaml
proxy_minion:
    Line 1.
    
    Line 3.
    Line 5.
    Line7.Line 9.
```    


The if statement and testing strings for conditions
===================================================


Something I do very oten is test strings retrieved from the pillar or grain interface for a condition. The following expressions are what I use most often:

```
{% raw %}{% set hostname = 'ar.core.ams01' %}
{{ 'ar' in hostname }}
{{ 'bar' in hostname }}
{{ 'bar' not in hostname }}
{{ hostname.startswith('ar') }}
{{ hostname.endswith('ams01') }}
{{ hostname.endswith('ams03') }}{% endraw %}
```

When we render the above template, we get the following returned:

```yaml
proxy_minion:
    True
    False
    True
    True
    True
    False
```

Here we see the expressions evaluate to `True` or `False`. We can combine these expressions with an `if` statement to show or hide sections of the template. Let's look at the following example:

```
{% raw %}
{%- set hostname = 'ar.core.ams01' -%}
{% if 'ar' in hostname -%}  
We found 'ar' in the hostname, configure something relevant to an 'ar'.
{% endif %}
{% if hostname.endswith('ams01') -%} 
The hostname ends with 'ams01', configuring something relevant to ams01.
{% endif %}
{% endraw %}
```

When we render this, we get the following:

```yaml
proxy_minion:
    We found 'ar' in the hostname, configure something relevant to an 'ar'.
    
    The hostname ends with 'ams01', configuring something relevant to ams01.
```


Conditional statements
======================


Let's assume that we have different device types in our network and that this information is stored as a grain. In the following example, through the use of conditional statements, we put all the different configurations into 1 template:

```
{% raw %}{%- set type = grains.facts.get('type') -%}     

{%- if type == 'cbr' -%}                        
set configuration for the cbr
{%- elif type in [ 'srr', 'xrr', ] -%}                        
set configuration for the srr or xrr
{%- else -%}                                    
set configuration for all the other roles
{%- endif -%}{% endraw %}
```
In the preceding template, we start out retrieving a grain value. After this, we check if the grain is equal to `cbr`. If it is, that configuration is shown. If it is not, we move on to the next test where we check if the type is present in a list. If it is, that configuration is displayed. If it is not, the `else` reveals what configuration will be applied to all other device types.

The previous example shows a few basic checks, but we can also test for multiple conditions at once. This could look something like this:

```
    {% raw %}{% if ( software_version in allowed_versions or allowed_versions == 'all' ) and
          ( model in allowed_models or allowed_models == 'all' ) and
          ( type in allowed_types or allowed_type == 'all' ) %}{% endraw %}
```


Another thing I use conditionals for in my templates is to have grains or pillar data determine the value a variable has. For instance, on Juniper MX and QFX, the configuration statement for LACP differs. In the below example, we fetch the model from the grains and have that decide the value of the variable:

```
{% raw %}{%- set model = grains.facts.get('model') -%}

{%- if 'qfx' in model -%}
    {% set ether_options_cfg = 'ether-options' %}
{%- elif 'mx' in model -%}
    {% set ether_options_cfg = 'gigether-options' %}
{%- endif -%}

set interfaces et-0/0/35 {{ ether_options_cfg }} 802.3ad ae0{% endraw %}
```

With things set up like this, we can render the template against MX as well as QFX. For example, when we render the template against a QFX device, we get this:
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
{% raw %}{% for n in range(5) %}
interface eth{{ n }}
 description unused
 switchport access vlan 999
 shutdown
{% endfor %}{% endraw %}
```

Using `salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2 default_renderer='jinja'`, we can see that the template renders as follows:

```yaml
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


String slicing
==============


We can slice strings to work with the part of the string we are interested in. Supose we have information encoded in the device name. Using string slicing, we can access the first and the last part of the string like so:

```
{% raw %}{% set hostname = 'ar.core.ams01' %}
{{ hostname[:2] }}
{{ hostname[-5:] }}{% endraw %}
```

This would render as follows:

```yaml
proxy_minion:
    ar
    ams01
```    

When we slice a string, we slice it by referencing the start, stop and stride of the characters in the string. Let's extract `word` from the following example:
```
{% raw %}{% set string = 'floousntd__' %}
{{ string[1:8:2] }}
{{ string[0:10:2] }}{% endraw %}
```

When we render this, we get the following:

```yaml
proxy_minion:
    lost
    found
```  


Splitting a string
==================

You can use `split` to divide a string into items in a list. You can then access the item that you are interested in. 

In the following example, `split` will be used on the string `192.168.1.1/24`. First, we'll print out the list of items created by `split`. After this, we access each individual item:

```
{% raw %}{% set ip_address = '192.168.1.1/24' %}
{{ ip_address.split('/') }}
{{ ip_address.split('/')[0] }}
{{ ip_address.split('/')[1] }}{% endraw %}
```

When we render the above template, we get the following:

```yaml
proxy_minion:    
    ['192.168.1.1', '24']
    192.168.1.1
    24
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
{% raw %}{%- for key, value in pillar.get('some_dict').items() -%} 
{{ key }} {{ value }}
{% endfor %}{% endraw %}
```

Endless possibilities here, but one of the things worth mentioning is passing a nested dictionary to a child template like so:
```
{% raw %}{% for nested_dict in pillar.get('nested-dict').values() -%}
{% include 'templates/some_template.j2' %}
{% endfor -%}{% endraw %}
```

We will be able to access the nested dictionary in `some_template` like this:
```
{% raw %}{{ nested_dict.some_key }}{% endraw %}
```

This will make it possible to use the dictionary as an instruction to generate the configuration for a service. You can, for instance, attach an inline pillar when calling a state through the API and use the dictionary to pass whatever you want to the template.



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
{% raw %}{% import_yaml '/var/tmp/example.yaml' as example_yaml %}


{% for interface, d in example_yaml | dictsort %}
interface {{ interface }}
 description {{ d.description }}
 switchport access vlan {{ d.vlan }}
{% endfor %}{% endraw %}

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


Using grains to perform a lookup in the pillar
==============================================


Imagine having a key called `datacenters` in the pillar that contains a mapping between the AS numbers you are using and your datacenters:

```yaml
datacenters:
  ams:
    65001
  tok:
    65002
  syd:
    65003
```

Supose that the datacenter in which the node is active, is stored as a grain value. We can retrieve the grain value and use it to perform a lookup in the pillar to grab the AS number the node should be configured with. 

In the next example, we first retrieve the grain value of the datacenter the node is in. Then, we use that variable to perform a lookup in the pillar:

```
{% raw %}{% set dc = grains.facts['datacenter'] %}
{% set as = pillar.datacenters.public[dc] %}
set routing-options autonomous-system {{ as }}{% endraw %}
```

When we render that against a proxy minion that has the datacenter grain value set to `syd`, the template would render as follows:

```
set routing-options autonomous-system 65003
```

Using grains or pillar data to include other files into the template
====================================================================


To keep things manageable, you will eventually start working with child templates. But it can happen that after a while, even the child templates grow into thousands of lines. 

Let’s suppose that a prefix list differs per device type. The following is an easy way to create a separate prefix-list template for every device while inheriting them in the same ‘master’ template:
```
{% raw %}{%- set type = grains.facts.get('type') -%}
{%- if type in [ 'brr', 'adr', 'xrr', ] %}
{% include 'templates/'~type~'_prefix_list.set' %}
{%- endif -%}{% endraw %}
```

When we render this template, the device type is fetched from the grains. After this, that variable is used to determine what template to include. If we have 30 devices types, we simply make 30 files that contain the appropriate prefix-lists and ensure that the template filename contains the device type. This beats having 30 elif’s. 

And perhaps some device types do not need any additional prefix-lists. By testing the device type against a list of device types, we prevent the template from failing to try and include a file that does not exist.



Debugging the template
======================


The error messages you’ll run into when templates break are not always that helpful. This can be tricky when you are dealing with large and complex templates or templates you have not touched in a while. 

It is nice to know that it is possible to make Salt generate a message that will end up in the proxy log. Observe the following template:

```
{% raw %}{%- set no_problem_here = 'just a var' -%}
{%- do salt.log.warning('debugging jinja 1: made it to line 2') -%} 
..
{%- do salt.log.warning('debugging jinja 2: In the loop that starts at line 23') -%}
..
{%- set erhm= pillar.get('lala').get('wow:wew') -%}
..
{%- do salt.log.warning('debugging jinja 3: got to the inner loop at line 32') -%}{% endraw %}
```

Let’s render the template using `salt proxy_minion slsutil.renderer salt://templates/my_first_template.j2`:
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
{% raw %}{% set interface = 'et-0/0/1' %}
{% set interface_description_dict = salt['junos.rpc']('get-interface-information', interface_name = interface, descriptions = True ) %}
{% set interface_description = interface_description_dict.get('rpc_reply').get('interface-information').get('physical-interface').get('description') %}

{{ interface_description }}{% endraw %}
 ```

When we render the template against a device that has `example description` configured as the description for `et-0/0/1`, we get the following:

```yaml
proxy_minion:
    interface description
```

Another use case that I ran into was when I was dealing with excess configuration. For instance, with some vendors, you can only delete syslog servers for which you specify the IP address that is configured: `no logging host 10.1.0.1`. But how can you template that without knowing what configuration is applied to your devices?
 
What you can do is retrieve the existing configuration from the device in order to be able to delete it:

```
{% raw %}{% set delete_existing = salt['em.nuke'](cmd='show running-config section ^logging', all=True) %}
{{ delete_existing }}{% endraw %}
```
Or
```
{% raw %}{{ salt['em.nuke'](section='logging') }}{% endraw %}
```

After rendering it using `salt proxy_minion slsutil.renderer path='salt://templates/my_first_template.j2'`, we get the following:

```yaml
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
{% raw %}salt proxy_minion state.apply states.example pillar='{"ops": { "interface" : "et-0/0/1", "change" : "666", }}'{% endraw %}
```

Inside the template, you can access this pillar data in the same way that you would access regular pillar data:
```
{% raw %}{% set interface = pillar.get('ops').get('interface') %}{% endraw %}
```
I have encountered multiple reasons for wanting to attach inline pillar data. One reason was to enable users to pass arguments to a state they are running.  Another reason was when I was using the Enterprise API. I found that passing a dictionary to a state is a very easy and neat way to have an external script pass data to templates.



Import other templates with context
===================================


If you have a ton of variables you are using in every template, it might be nice to know that you can dump them all in a single template and include that template elsewhere.

For instance, let’s create a template called default:
```
{% raw %}{%- set type = grains.facts.get('type') -%}
{%- set software_version = grains.facts.get('software-version') -%}
{%- set model = grains.facts.get('model') -%}
{%- set password_information = pillar.get('secret_data') -%}{% endraw %}
```

We can import this in other templates like this:
```
{% raw %}{%- import 'templates/default.j2' as example with context -%}
{{ example.model }}{% endraw %}
```
Since we import the template as `example`, whenever we access a variable set in that template, we prefix it with `example.`.
The main advantages are that it keeps the templates smaller and the default file is easy to maintain in case something changes or needs to be added to multiple templates.

Additionally, another thing worth noting is that child templates have access to the variables declared in the parent template. So if we import a default template to be able to use variables everywhere, we will also be able to use those in the child templates as well.



Using Jinja in state and pillar files
=====================================


In addition to using jinja in templates, you can also use it in other places, like in your pillar and in your state files. 

The following is an example where doing something in a state file depends on whether or not ‘xyz’ is found in the device name:

```
{% raw %}{%- set device_name = pillar.get('device-name') -%}

{%- if 'xyz' in device_name -%}

generate_something:
  cmd.script:
    - name: /usr/bin/python
    - source: salt://utility/some_script.py
    - args: "{{ pw }}"

{%- endif %}{% endraw %}
```



Salt has some pretty good additional extensions
===============================================


It is worth familiarizing yourself with the Jinja extensions SaltStack offers as many of them can come in quite handy. Reading about them before starting your templating efforts might help you a lot.

Some of the extensions allow you to do pretty cool things. For example, test if something is an IP address:
```
{% raw %}{{ '192.168.0.1' | is_ip }}{% endraw %}
```
Using regex replace:
```
{% raw %}{% set my_text = 'yes, this is a TEST' %}
{{ my_text | regex_replace(' ([a-z])', '__\\1', ignorecase=True) }}{% endraw %}
```
Raising custom errors:
```
{% raw %}{{ raise('Custom Error') }}{% endraw %}
```

These are just some examples. No point in me covering all of them, just read up on the rest right here: [understanding Jinja](https://docs.saltstack.com/en/latest/topics/jinja/index.html)

In case you just want to read up on Jinja, you can check out this site: [Jinja2 doc](http://jinja.pocoo.org/docs/)



Wrapping up
===========


These were some of the tips and examples that I really could have used when I started templating in Salt myself. I hope this helps clarify a few things and that you have gained some insights.

If you have any additional tips, let me know via mail. Thank you!
