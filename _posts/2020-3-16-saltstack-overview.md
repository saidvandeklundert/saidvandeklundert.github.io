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

SaltStack comes with its own set of terms and terminolgy that can be overwhelming at first. The following picture captures part of the architecture and displays a lot 'Saltspeak':

{:refdef: style="text-align: center;"}
![SaltStack architecture](/img/saltstack-architecture.png "SaltStack architecture")
{: refdef}

**Salt-master**: the Salt master service is what manages and orchestrates the remote systems. Systems that are managed by the salt master service can be managed through a variety of ways:
- **Salt-minion**: in this case, the salt-minion process is running on the system that the salt-master is managing. The minion can run on (almost) any system that allows for a Python interpreter.
- **Proxy-minion**: for systems that do not allow you to run a salt-minion process or devices that are not capable of letting you (old networking devices for example), you can run the proxy-minion process as an intermidiary between the master and the device. The salt-master will commicate with the proxy over the event bus (the salt way) and the proxy can translate the instructions received from the master to whatever the device understands ( using an SSH channel, an API, etc.) 
- **Salt-ssh**: an agentless based approach to control another system. All that is required on the other system is for SSH to be running. This approach does not scale as well and works differently in the sense that the message bus is not utilized. 

**Message bus**: Salt uses the ZeroMQ message bus to facilitate communication between the master and the minions by default. This open source software is a high-speed messaging bus that carries messages between the master and the (proxy-) minions. Both master as well as minions can utitlize this message bus to send and receive messages for various purposes. 

Some examples of what the message bus is used for:
- the master can use the message bus to issue a command to one, several or all minions
- a minion can generate an event to signal to the master or another minion that something has happened

The message bus is sometimes touted as one of the greatest strengths of the Salt framework. This is because it serves as a construct on which other features in Salt are built. Salt’s ability to scale, to orchestrate or to be event-driven for instance all rely on this message bus. It is also something that sets Salt apart from several other automation frameworks.
Two Salt functionalities that are particularly related, or intertwined, with the message bus are the reactor and the beacon.

**Reactor**: The reactor is a master-side interface that is used to watch the event bus for messages and respond to them. A reactor can be made to respond to a pattern by means of starting a state for example. In turn, the state systems offers you a lot of freedom to define, or code, whatever reponse you think is appropriate.

**Beacon**: Salt beacon is a minion-side feature that can be used to generate events that are then send on to the message bus. A beacon can be used for anything on the system that the minion controls or has access to. For instance, a beacon can be used to generate an event when a file is changed or when a network interface goes down. You could then use a reactor to determine if and what actions should follow asa result of a beacon.

**SaltStack data interfaces**: SaltStack offers remote execution, configuration management as well as orchestration. And one of the nice things about Salt is the way it can be made to use a varitiety of data interfaces in the same way. In general, there are 3 different data interfaces inside Salt:
- Pillar: pillar data is generated by the master. The pillar data, or a part of it, is something the master shares with it’s (proxy-) minions based on the way it is configured. Pillar data is encrypted on a per-minion basis. Due to the fact that pillar data is encrypted, it becomes ‘expensive’ resource-wise. For small scale deployment, the pillar can be used to store all sorts of properties that describe the infrastructure as a whole (NTP server, DNS servers, etc.). For large scale deployments, it is a good idea to limit pillar data to sensitive information like API tokens and passwords only and turn to other data interfaces for the more generic properties of the infrastructure.
- Grains: an interface that describes the system that is being managed. Every (proxy-) minion generates grains data that describe the (static) properties of the system that is under it’s control. Some example of grains data are: OS type, RAM, CPU type, IP addresses and interfaces, etc.
- External pillar: Salt can be configured to retrieve data from an external sytem, and expose that data as pillar data. An example would be where someone uses Hashicorp Vault to store sensitive information. The Salt master can be made to use Vault and make an API call to retrieve all passwords/API tokes when building the pillar data for it’s minions, allowing you to extend Salt with other systems.

In addition to these data interfaces, there is also another way in which you can have Salt use data from other sources. This is helpful in case the grains, pillar and external pillar do not suffice. The execution module is a very powerfull way to make a master or minion talk to another system and expose data from that system to Salt in any way you like. One example would be where you write an execution module (a Python script ) to perform a SQL query to retrieve data. You could opt to use that in a state or template directly. Alternatively,  In case you have thousands of minions, you can also choose to store this data as JSON, YAML or a map file and include it in your Jinja later on. This way, you would fetch the data once and still allow Salt to use it everywhere.


**SaltStack Enterprise**: a commercial product that includes:
- Centralized control of your salt masters through a GUI or an API
- Role based access control
- Multi-master support
- LDAP integration
- Reporting


https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/saltstack/_posts/2020-3-16-saltstack-overview.md