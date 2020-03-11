---
layout: post
title: SaltStack overview 
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

Diving into SaltStack for the first time can be difficult and confusing. There is a lot of terminology involved that is going to be new. In addition to that, it is not immediately apparent what parts there are to SaltStack.

This article aims to provide an overview of Salt and all of the different terminologies and constructs that come with it. Instead of providing an exhaustive list of possibilities and knobs that exist in Salt, the aim is to give you an idea of what the SaltStack framework has to offer and what most of the individual constructs could be used for. 



## What SaltStack does


### Remote execution:

Salt is a remote execution framework that can be used to manage thousands of systems. These systems can be Linux servers, Docker containers, networking devices and more.

### Configuration management:

Salt is very well suited to act as a configuration management system that can be used against a plethora of systems. Some of the reasons that make it suitable as a configuration management system are the following:
- Salt can easily interface with any other external system
- Salt offers you many ways to plug in your Python/Jinja to overcome and deal with any configuration challenge
- Salt offers granular control as well as ways to abstract complexity and work on a different layer using the Salt state system

### Orchestration and automation:

SaltStack’s capabilities for remote execution and configuration management can be further extended with the ability to orchestrate the order in which events are to take place. This can be an orchestrated series of events that is initiated whenever a user runs an orchestration state, or a certain series of events that take place as a reaction to something happening on a system.

An example of what you could orchestrate with Salt is the following:
- Extend an IP-Fabric to a Linux server
- Install Docker engine on said server
- Start different containers that offer microservices in a specific order
- Update the firewall to account for the newly provisioned services


### Salt concepts and architecture overview.

{:refdef: style="text-align: center;"}
![SaltStack architecture](/img/saltstack-architecture.png "SaltStack architecture")
{: refdef}



https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/saltstack/_posts/2020-3-16-saltstack-overview.md