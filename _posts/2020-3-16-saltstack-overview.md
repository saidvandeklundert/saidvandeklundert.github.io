---
layout: post
title: SaltStack overview 
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

Diving into SaltStack for the first time can be difficult and confusing. There is a lot of terminology involved that is going to be new. So much so, that it makes it difficult to get a clear overview of how everything is working together and how the different parts might be useful to you.

This article aims to provide an overview of Salt and all of the different terminologies and constructs that come with it. Instead of providing an exhaustive list of possibilities and knobs that exist in Salt, the aim is to give you an idea of what the SaltStack framework has to offer and what most of the individual constructs could be used for. 

First though, let's see what SaltStack can do for an organization as a whole.

## What SaltStack does


### Remote execution:

Salt is a remote execution framework that can be used to manage thousands of systems. These systems can be Linux servers, Docker containers, networking devices and more.

### Configuration management:

Salt is well suited to act as a configuration management system. Some of the reasons that make it suitable as a configuration management system are the fact that Salt:
- can interface with a plethora of systems (Linux, Windows, Juniper, Cisco, ESXi and more)
- can be made to interface with other external systems
- offers you easy ways to plug in Python libraries to overcome configuration challenge
- offers granular control as well as ways to abstract complexity and work on a different layers

### Orchestration and automation:

SaltStack’s capabilities for remote execution and configuration management can be further extended with the ability to orchestrate the order in which events are to take place. This can be an orchestrated series of events that is initiated whenever a user runs an orchestration state, or a certain series of events that take place as a reaction to something happening on a system.

An example of what you could orchestrate with Salt is the following:
- Extend an IP-Fabric constisting of Juniper and Arista switches to a Linux server
- Install Docker engine on said server
- Start different containers that offer microservices in a specific order
- Update the firewall to account for the newly provisioned services



## Salt concepts and architecture overview.

SaltStack comes with its own set of terms and terminolgy that can be overwhelming at first. The following picture captures part of the architecture and displays a lot 'Saltspeak':

{:refdef: style="text-align: center;"}
![SaltStack architecture](/img/saltstack-architecture.png "SaltStack architecture")
{: refdef}

Let's go over some of this Saltspeak and provide a high-level description on all the different parts there are to this architecture.

## Salt-master

The Salt master service is what manages and orchestrates the remote systems. Different methods exist to ensure the Salt master is redundant (multimaster, failover master and more). The master has a several constructs at its disposal to control the remote systems. Some of these constructs, discussed in more detail later on, are the following:
- states and orchestration states
- execution modules
- runners

Systems that are managed by the salt master service can be managed through a variety of ways:
- minions
- proxy-minions
- ssh
 
When minions or proxy-minion are used, communication between the master and the minions is facilitated through a <b>message bus</b> called <b>ZeroMQ</b>.  

{:refdef: style="text-align: center;"}
![SaltStack master and minions](/img/salt_master_and_minion.png "SaltStack master and minions")
{: refdef}


## Salt-minion

In this case, the salt-minion process is running on the system that the salt-master is managing. The minion can run on (almost) any system that allows for a Python interpreter. It will communicate with the master service over the event bus.

{:refdef: style="text-align: center;"}
![SaltStack minion](/img/salt_minion.png "SaltStack minion")
{: refdef}


## Proxy-minion

For systems that do not allow you to run a salt-minion process, you can run the proxy-minion process as an intermidiary between the master and the device. The salt-master will communicate with the proxy over the event bus. The proxy process will communicate with the device using whatever method the device understands. This can be an SSH channel, an API, etc. 

{:refdef: style="text-align: center;"}
![SaltStack proxy-minion](/img/salt_proxy_minion.png "SaltStack proxy-minion")
{: refdef}

The proxy-minion process that is used to control a system does not have to be running on the same server/container that is housing the master service.

## Salt-ssh

An agentless based approach to control another system. All that is required on the other system is for SSH to be running. Though in some cases convenient, this approach does not scale as well. Using salt-ssh will put a lot of load on the master. And since there is no message bus in between the master and the system that is being managed with salt-ssh, the master cannot control a lot of systems at the same speed.

{:refdef: style="text-align: center;"}
![SaltStack Salt-ssh](/img/salt_ssh.png "SaltStack Salt-ssh")
{: refdef}


## Message bus

Salt uses the ZeroMQ message bus to facilitate communication between the master and the minions by default. This open source software is a high-speed messaging bus that carries messages between the master and the (proxy-) minions. Both master as well as minions can utitlize this message bus to send and receive messages for various purposes. 

Some examples of what the message bus is used for:
- the master can use the message bus to issue a command to one, several or all minions
- minions utilize the message bus to return the results of any job that was executed
- minions can generate an event to signal to the master that something has happened

The message bus is sometimes touted as one of the greatest strengths of the Salt framework. This is because it serves as a construct on which other features in Salt are built. Salt’s ability to scale, to orchestrate or to be event-driven for instance all rely on this message bus. It is also something that sets Salt apart from several other automation frameworks.
Two Salt functionalities that are particularly related, or intertwined, with the message bus are the reactor and the beacon.

## SalStack reactors and beacons

Reactors and beacons are the constructs that can be leveraged to make the master respond to events generated by the minion.

{:refdef: style="text-align: center;"}
![SaltStack reactors and beacons](/img/salt_reactor_and_beacon.png "SaltStack reactors and beacons")
{: refdef}

**Beacon**: Salt beacon is a minion-side feature that can be used to generate events that are then send on to the message bus. A beacon can be used for anything on the system that the minion controls or has access to. For instance, a beacon can be used to generate an event when a file is changed or when a network interface goes down. You could then use a reactor to determine if and what actions should follow as a result of a beacon. The beacon is send on a port that only the master is listening to (4506).

**Reactor**: The reactor is a master-side interface that is used to watch the event bus for messages and respond to them. A reactor can be made to respond to a pattern, or tag. The response can be to start a state or a runner.


## SaltStack data interfaces

SaltStack offers remote execution, configuration management as well as orchestration. And one of the nice things about Salt is the way it can be made to use a varitiety of data interfaces in the same way. In general, there are 3 different data interfaces inside Salt:
- Pillar: pillar data is generated by the master. The pillar data, or a part of it, is something the master shares with it’s (proxy-) minions based on the way it is configured. Pillar data is encrypted on a per-minion basis. Due to the fact that pillar data is encrypted, it becomes ‘expensive’ resource-wise. For small scale deployment, the pillar can be used to store all sorts of properties that describe the infrastructure as a whole (NTP server, DNS servers, etc.). For large scale deployments, it is a good idea to limit pillar data to sensitive information like API tokens and passwords only and turn to other data interfaces for the more generic properties of the infrastructure.
- Grains: an interface that describes the system that is being managed. Every (proxy-) minion generates grains data that describe the (static) properties of the system that is under it’s control. Some example of grains data are: OS type, RAM, CPU type, IP addresses and interfaces, etc.
- External pillar: Salt can be configured to retrieve data from an external sytem, and expose that data as pillar data. An example would be where someone uses Hashicorp Vault to store sensitive information. The Salt master can be made to use Vault and make an API call to retrieve all passwords/API tokes when building the pillar data for it’s minions, allowing you to extend Salt with other systems.

{:refdef: style="text-align: center;"}
![SaltStack external pillar](/img/salt_external_pillar.png "SaltStack external pillar")
{: refdef}

In addition to these data interfaces, there is also another way in which you can have Salt use data from other sources. This is helpful in case the grains, pillar and external pillar do not suffice. The execution module is a very powerfull way to make a master or minion talk to another system and expose data from that system to Salt in any way you like. One example would be where you write an execution module (a Python script ) to perform a SQL query to retrieve data. You could opt to use that in a state or template directly. Alternatively,  In case you have thousands of minions, you can also choose to store this data as JSON, YAML or a map file and include it in your Jinja later on. This way, you would fetch the data once and still allow Salt to use it everywhere.


## SaltStack Enterprise (SSE)

SSE is a commercial product that includes:
- Centralized control of your salt masters through a GUI or an API
- Role based access control
- Multi-master support
- LDAP integration
- Reporting


## States

A Salt state is a collection of actions you want to perform to put a system into a certain ‘state’. Inside the state, you can call different execution modules, custom execution modules and/or custom states to bring a system into a certain ‘state’ (hence the name): 

{:refdef: style="text-align: center;"}
![Salt State](/img/salt_state.png "Salt State")
{: refdef}

A salt-master can tell a minion that it needs to run a state. The minion will execute the state and return the result using the ZeroMQ message bus. Because of this, it is possible to run a state on many thousands of systems at the same time without overwhelming the master.

The state system has access to most Salt interfaces. This means that from within a state, you are free to access the pilllar, the grains and other Salt facilities. Additionally, the state system itself can be used by other Salt features as well. Salt reactors and runners for instance can also run states. Obviously, it is also possible to run a state from the CLI at will or to schedule states at regular intervals.

Another nice thing about the state system is that it allows you to use requisites to create relationships between different states and/or actions inside a single state. An example of what this would enable you to do is to make sure an action is executed only if another action was completed succesfully.  

Though other options exist, most Salt states are written in YAML and Jinja. This makes the state easy to read and a nice abstraction layer to manage complexity.

The general phylisophy is that a state should be idempotent. Regardless of how many times you run a state, it should bring a system into the same state, always.

## Execution modules

you like Python, the (custom)-execution module will make you fall in love with Salt. The execution module is a Python script with functions that you can call on the command line or use inside a Salt state. 

{:refdef: style="text-align: center;"}
![Salt execution module ](/img/salt_execution_module.png "Salt execution module ")
{: refdef}

Salt comes with several execution modules out of the box. Those modules can be found <a href="https://github.com/saltstack/salt/tree/master/salt/modules" target="_blank">here</a>. This is also a nice folder to browse in case you are looking to get started writing your own execution module. 

Writing your own is something I would really recommend. An execution module you produce yourself is referred to as a 'custom execution module'. Some of the things I really like about the custom execution module are the following:
- you can express yourself in Python 
- you can use the standard library and, if the system the minion runs on let's you, pip install whatever you want
- Salt interfaces, like the pillar and grains, are available through special dunder methods.
- the execution module can be used in other Salt facilities as well, like in states for instance or even other execution modules
- the master can utilize the message bus to have all minions run the execution module in (near) parralel
 

## Custom states

A custom state is a construct that Salt offers you to write State functions in. You will be able to call these functions directly in the state system. The custom state is written in Python and it can interface with other Salt features. Some of the things you can do inside a custom state functions are:
- Interface with pillar or grains data
- Cross call (custom) execution modules

The general idea is that the execution module is a low level function that does 1 thing while the custom state is used to put in additional logic. 

Unlike (custom-) execution modules, custom state functions cannot be called from the CLI. 

## Runners

Salt runners are similar to (custom) execution modules. In a salt-runner, the Python you write will have access to all the available Salt interfaces from the master perspective. 

The key difference between an execution module and a runner is the fact that a runner is executed on the salt-master instead of on the salt-minion.  Since the runner function is executed on the master, there is the possibility to have it perform actions on minions. 

{:refdef: style="text-align: center;"}
![SaltStack runner](/img/salt_runner_example.png "SaltStack runner")
{: refdef}

## Orchestration

Orchestration states are run on the master and offers a means to ‘orchestrate’ a series of events to take place. 

{:refdef: style="text-align: center;"}
![SaltStack orchestration state](/img/salt_orchestration_state.png "SaltStack orchestration state")
{: refdef}


Orchestration states are a collection of actions that are executed on the master. Some of actions that you can make the master perform are the following:
- call execution modules or states on minions
- instruct minions to run states
- make the master execute a salt-runner
- use salt-ssh to perform a task on a remote system
- call, or follow up with additional orchestration states




Obviously, a lot more is possible.
https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/saltstack/_posts/2020-3-16-saltstack-overview.md