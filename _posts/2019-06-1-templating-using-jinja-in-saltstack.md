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




