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

{:refdef: style="text-align: center;"}
![SaltStack master and minions](/img/salt_master_and_minion.png "SaltStack master and minions")
{: refdef}

- **Salt-minion**: in this case, the salt-minion process is running on the system that the salt-master is managing. The minion can run on (almost) any system that allows for a Python interpreter.
- **Proxy-minion**: for systems that do not allow you to run a salt-minion process or devices that are not capable of letting you (old networking devices for example), you can run the proxy-minion process as an intermidiary between the master and the device. The salt-master will commicate with the proxy over the event bus (the salt way) and the proxy can translate the instructions received from the master to whatever the device understands ( using an SSH channel, an API, etc.) 
- **Salt-ssh**: an agentless based approach to control another system. All that is required on the other system is for SSH to be running. This approach does not scale as well and works differently in the sense that the message bus is not utilized. 

**Message bus**: Salt uses the ZeroMQ message bus to facilitate communication between the master and the minions by default. This open source software is a high-speed messaging bus that carries messages between the master and the (proxy-) minions. Both master as well as minions can utitlize this message bus to send and receive messages for various purposes. 

Some examples of what the message bus is used for:
- the master can use the message bus to issue a command to one, several or all minions
- a minion can generate an event to signal to the master or another minion that something has happened

The message bus is sometimes touted as one of the greatest strengths of the Salt framework. This is because it serves as a construct on which other features in Salt are built. Salt’s ability to scale, to orchestrate or to be event-driven for instance all rely on this message bus. It is also something that sets Salt apart from several other automation frameworks.
Two Salt functionalities that are particularly related, or intertwined, with the message bus are the reactor and the beacon.

**SalStack reactors and beacons**:

Reactors and beacons are the constructs that can be leveraged to make the master respond to events generated by the minion:

{:refdef: style="text-align: center;"}
![SaltStack reactors and beacons](/img/salt_reactor_and_beacon.png "SaltStack reactors and beacons")
{: refdef}

**Reactor**: The reactor is a master-side interface that is used to watch the event bus for messages and respond to them. A reactor can be made to respond to a pattern, or tag. The response can be to start a state or a runner.

**Beacon**: Salt beacon is a minion-side feature that can be used to generate events that are then send on to the message bus. A beacon can be used for anything on the system that the minion controls or has access to. For instance, a beacon can be used to generate an event when a file is changed or when a network interface goes down. You could then use a reactor to determine if and what actions should follow as a result of a beacon. The beacon is send on a port that only the master is listening to (4506).


**SaltStack data interfaces**: SaltStack offers remote execution, configuration management as well as orchestration. And one of the nice things about Salt is the way it can be made to use a varitiety of data interfaces in the same way. In general, there are 3 different data interfaces inside Salt:
- Pillar: pillar data is generated by the master. The pillar data, or a part of it, is something the master shares with it’s (proxy-) minions based on the way it is configured. Pillar data is encrypted on a per-minion basis. Due to the fact that pillar data is encrypted, it becomes ‘expensive’ resource-wise. For small scale deployment, the pillar can be used to store all sorts of properties that describe the infrastructure as a whole (NTP server, DNS servers, etc.). For large scale deployments, it is a good idea to limit pillar data to sensitive information like API tokens and passwords only and turn to other data interfaces for the more generic properties of the infrastructure.
- Grains: an interface that describes the system that is being managed. Every (proxy-) minion generates grains data that describe the (static) properties of the system that is under it’s control. Some example of grains data are: OS type, RAM, CPU type, IP addresses and interfaces, etc.
- External pillar: Salt can be configured to retrieve data from an external sytem, and expose that data as pillar data. An example would be where someone uses Hashicorp Vault to store sensitive information. The Salt master can be made to use Vault and make an API call to retrieve all passwords/API tokes when building the pillar data for it’s minions, allowing you to extend Salt with other systems.

{:refdef: style="text-align: center;"}
![SaltStack external pillar](/img/salt_external_pillar.png "SaltStack external pillar")
{: refdef}

In addition to these data interfaces, there is also another way in which you can have Salt use data from other sources. This is helpful in case the grains, pillar and external pillar do not suffice. The execution module is a very powerfull way to make a master or minion talk to another system and expose data from that system to Salt in any way you like. One example would be where you write an execution module (a Python script ) to perform a SQL query to retrieve data. You could opt to use that in a state or template directly. Alternatively,  In case you have thousands of minions, you can also choose to store this data as JSON, YAML or a map file and include it in your Jinja later on. This way, you would fetch the data once and still allow Salt to use it everywhere.


**SaltStack Enterprise**: a commercial product that includes:
- Centralized control of your salt masters through a GUI or an API
- Role based access control
- Multi-master support
- LDAP integration
- Reporting

**States**: a Salt state is a collection of actions you want to perform to put a system into a certain ‘state’. Though other options exist, most Salt states are written in YAML and Jinja. This makes the state easy to read and manage.

Inside the state, you can call different execution modules, custom execution modules and/or custom states to bring a system into a certain ‘state’. 
The state system has to most Salt interfaces. So from within a state, you are free to access the pilllar, or grains, etc. Additionally, the state system can be used by a lot of other Salt facilities as well. For instance, a Salt reactor can call a state in response to an event.

States can be invoked at will, they can be made to run at (minion) startup and they can be scheduled at regular intervals. Additionally, the state system allows you to use requisites to create relationships between different states and/or actions inside a single state.  

The general phylisophy is that a state should be idempotent. Regardless of how many times you run a state, it should bring a system into the same state, always.

**Execution modules**: if you like Python, the (custom)-execution module will make you fall in love with Salt. The execution module is a Python script with functions that you can call on the command line or use inside a Salt state. 

Inside the execution, you are able to use Salt-interfaces by calling various dunder-methods. You will, for instance, be able use pillar, grains and other execution modules in your script. 

Apart from this, it is also possible to import ‘regular’ Python modules. You do have to ensure that whatever extra module you are using is available on the system that needs to run the execution module.

Salt comes with several execution modules out of the box and a nice thing to do in order to get started writing custom execution modules is to look for inspiration at how the modules Salt is shipped with look: https://github.com/saltstack/salt/tree/master/salt/modules

In case you want to plug in your own scripts, you can write and create whatever you can dream up. You will need to place the script in the proper folder, which is `/srv/salt/_modules`. After this, the scripts inside those folder, dubbed custom execution modules, can be synched to the (proxy-) minions.  

**Custom states**: a custom state is a construct that Salt offers where you can write State functions in Python. You will be able to call these functions directly in the state system. 

The custom state functions can interface with other Salt features. Some of the things you can do inside a custom state functions are:
- Interface with pillar or grains data
- Cross call (custom) execution modules
The general idea is that the execution module is a low level function that does 1 thing while the custom state is used to put in additional logic. 
Unlike (custom-) execution modules, custom state functions cannot be called from the CLI. 

**Runners**: salt runners are similar to (custom) execution modules. In a salt-runner, the Python you write will have access to all the available Salt interfaces from the master perspective. 

The key difference between an execution module and a runner is the fact that a runner is executed on the salt-master instead of on the salt-minion.  Since the runner function is executed on the master, there is the possibility to have it perform actions on minions. 

{:refdef: style="text-align: center;"}
![SaltStack runner](/img/salt_runner_example.png "SaltStack runner")
{: refdef}

**Orchestration**: orchestration states are run on the master and offers a means to ‘orchestrate’ a series of events to take place. 

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