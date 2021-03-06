---
layout: post
title: SaltStack overview 
tags: [automation, saltstack]
image: /img/salt_stack_logo.jpg
---

Diving into SaltStack for the first time can be difficult and confusing. There is a lot of terminology involved that is going to be new. This makes it difficult to get a clear overview of how everything is working together and how the different parts might be useful to you.

{:refdef: style="text-align: center;"}
![SaltStack](/img/saltstack_logo_large.jpg  "SaltStack" )
{: refdef}

This article aims to provide an overview of Salt and all of the different terminologies and constructs that come with it. Instead of providing an exhaustive list of possibilities and knobs that exist in Salt, the aim is to give you an idea of what the SaltStack framework has to offer and what most of the individual constructs can be used for. 

First, I will cover what SaltStack does at a very high level. After this, most of the constructs that are part of the Salt architecture are discussed.

## What SaltStack has to offer


### Remote execution:

Salt is a remote execution framework that can be used to manage thousands of systems. These systems can be servers, containers, networking devices and more.

### Configuration management:

Salt gives you granular control as well as ways to abstract complexity and work on a different layers. On a practical level, Salt can interface with a plethora of systems. You can use SaltStack to manage Linux, Windows, Juniper, Cisco, ESXi and much more. Additionally, SaltStack can easily connect to other systems to pull in whatever data is needed for your configuration efforts. 

### Orchestration and automation:

Salt has the ability to orchestrate the order in which events are to take place. An example of what you could orchestrate with Salt is the following:
- Extend an IP-Fabric consisting of routers and switches to a Linux server
- Install Docker engine on said server
- Start different containers that offer microservices in a specific order
- Update the firewall to account for the newly provisioned services


## SaltStack architecture overview

SaltStack comes with its own set of terms and terminology. The following is a picture that captures part of the architecture, showing a lot of 'Saltspeak':


{:refdef: style="text-align: center;"}
![SaltStack architecture](/img/saltstack_architecture.png  "SaltStack architecture" )
{: refdef}

Perhaps a little overwhelming at first, but really great to work with after you wrap your head around it. High-level, the following are the core components of the SaltStack architecture:

- **Salt-master**: manages and orchestrates the remote systems
- **Salt-minion**: system that is being controlled by a Salt master
- **Proxy-minion**: process that acts as an intermediary between the master and the system that is being managed
- **Salt-ssh**: Salt feature that allows the salt-master to manage a system using SSH
- **ZeroMQ Message bus**: facilitates exchange of messages between the salt-master and the salt-minions / proxy-minions
- **SaltStack data interfaces**:
  - **pillar**: contains data generated by the master that is made available to some or all systems that are being managed
  - **grains**: interface to data generated by the minion that describes the system that is being managed
  - **external pillar**: interface to applications and constructs outside of Salt
- **States**: collection of actions that a minion or proxy-minion can be made to execute
- **Execution modules**: a Python script that can be executed on a minion or a proxy-minion
- **Custom states**: Python functions that you can call in a state
- **Runners**: a Python script that is executed on the master
- **Orchestration state**: a state that is executed on the master and that can call states and modules on minions as well as start a runner
- **Reactor**: master-side interface that can react to an event (generated by a beacon)
- **Beacon**: minion-side feature, used to generate an event

Let's provide a little more detail and context for all this Saltspeak.

### Salt-master

The Salt master service is what manages and orchestrates the remote systems. Different methods exist to ensure the Salt master is redundant (multimaster, failover master and more). The master has a several constructs at its disposal to control the remote systems. Some of these constructs, discussed in more detail later on, are the following:
- states and orchestration states
- execution modules
- runners

The Salt-master can manage systems in a variety of ways:
- minions
- proxy-minions
- salt-ssh
 
When minions or proxy-minion are used, communication between the master and the minions is facilitated through a <b>message bus</b> called <b>ZeroMQ</b>.  

{:refdef: style="text-align: center;"}
![SaltStack master and minions](/img/salt_master_and_minion.png "SaltStack master and minions")
{: refdef}


### Salt-minion

A system that is being controlled by a salt-master is called a <b>Salt-minion</b>. Systems under the control of a salt-master run a salt-minion service. The salt-minion service can run on (almost) any system that allows for a Python interpreter. 

{:refdef: style="text-align: center;"}
![SaltStack minion](/img/salt_minion.png "SaltStack minion")
{: refdef}

The salt-minion service and the salt master communicate using the event bus. The minion will execute whatever assignment it receives and when the minion has completed its task, it will inform the master of the results.

### Proxy-minion

For systems that do not allow you to run a salt-minion service, you can run the proxy-minion process as an intermediary between the master and the system that is being managed. The salt-master will communicate with the proxy over the event bus. The proxy process will communicate with the system being managed using a method that system will understand. This can be an SSH channel, an API, etc. 

{:refdef: style="text-align: center;"}
![SaltStack proxy-minion](/img/salt_proxy_minion.png "SaltStack proxy-minion")
{: refdef}

The proxy-minion process that is used to control a system does not have to be running on the same server that is used to run the master service. The <b>salt-proxy</b> process can be running on another server. Just make sure that server has IP connectivity with the master as well as the systems that it is managing:

{:refdef: style="text-align: center;"}
![SaltStack proxy-minion](/img/salt_proxy_minion_process.png "SaltStack proxy-minion")
{: refdef}

### Salt-ssh

An agentless based approach to control another system. All that is required on the other system is for SSH to be running. Though in some cases convenient, this approach does not scale as well. Using salt-ssh will put a lot of load on the master. And since there is no message bus in between the master and the system that is being managed with salt-ssh, the master cannot control as many systems at the same time.

{:refdef: style="text-align: center;"}
![SaltStack Salt-ssh](/img/salt_ssh.png "SaltStack Salt-ssh")
{: refdef}

An interesting use for salt-ssh is to install minion software on a system and properly configure that minion in order to bring it under the control of Salt.


### Message bus

Salt uses the ZeroMQ message bus to facilitate communication between the master and the minions by default. This open source software is a high-speed messaging bus that carries messages between the master and the (proxy-) minions. Both master as well as minions can utilize this message bus to send and receive messages for various purposes. 

Some examples of what the message bus is used for:
- the master can use the message bus to issue a command to one, several or all minions
- minions utilize the message bus to return the results of any job that was executed
- minions can generate an event to signal to the master that something has happened

The message bus is sometimes touted as one of the greatest strengths of the Salt framework. This is because it serves as a construct on which other features in Salt are built. Salt’s ability to scale, to orchestrate or to be event-driven for instance all rely on this message bus. It is also something that sets Salt apart from several other automation frameworks.


### SaltStack data interfaces

In general, there are 3 different data interfaces that can be identified in Salt:
- **Pillar**: an interface that can contain data for all minions. Pillar data is generated by the master and it is shared with its (proxy-) minions based on the way the master is configured. For small scale deployment, the pillar can be used to store all sorts of properties that describe the infrastructure as a whole (NTP server, DNS servers, etc.). But since pillar data is encrypted and generated by the master, for large scale deployments, it is a good idea to limit pillar data to sensitive information like API tokens and passwords. 
- **Grains**: an interface that describes the system that is being managed. Every (proxy-) minion generates grains data that describe the (static) properties of the system that is under its control. Some example of grains data are: OS type, RAM, CPU type, IP addresses and interfaces, etc.
- **External pillar**: Salt can be configured to retrieve data from an external system. An example would be where someone uses Hashicorp Vault to store sensitive information. The Salt master can be made to retrieve all passwords/API tokens from Vault when building the pillar data for its minions.

{:refdef: style="text-align: center;"}
![SaltStack external pillar](/img/salt_external_pillar.png "SaltStack external pillar")
{: refdef}

In addition to these data interfaces, there is also another way in which you can have Salt use data from other sources. This is the execution module, which is a Python script that can be used by various other Salt constructs. 

The execution module can be used to bring in data from other systems in case the grains, pillar and external pillar do not suffice.  One example would be where you write an execution module to perform a SQL query to retrieve data. You could opt to use that in a state or template directly. Alternatively, in case you have thousands of minions, you can also choose to store gathered data as JSON or YAML and include it in your template this way. 

If you are thinking about going this route, another nice construct worth pointing out is the map file. You can use YAML, JSON, Jinja or even Python in a map file. It can be included in states and templates and is not as taxing to the overall system as pillar data. This is because it does not have to be rendered by the master and it does not require any encryption. 

Storing data in JSON, YAML or map-files allow you to fetch the data once while enabling Salt to use it (almost) everywhere else. Recommended to look into as an alternative to pillar data for large scale deployments.


SaltStack offers remote execution, configuration management as well as orchestration. And one of the nicest things about the data interfaces in Salt is the fact that they can be used (almost) everywhere. 

Pillar and grains for instance can be used to:
- determine what systems to target
- enhance <a href="https://saidvandeklundert.net/2019-06-08-templating-for-network-engineers-in-saltstack/" target="_blank">templating</a> efforts for (networking-) device configuration
- make states and modules act differently based on their value


### States

A Salt state is a collection of actions you want to perform to put a system into a certain ‘state’ (hence the name). Inside the state, you can call different execution modules, states and/or custom state functions.

{:refdef: style="text-align: center;"}
![Salt State](/img/salt_state.png "Salt State")
{: refdef}

A salt-master can tell a minion that it needs to run a state. The minion will execute the state and return the result using the ZeroMQ message bus. Because the minion executes the state, it is possible to run a state on many thousands of systems at the same time without overwhelming the master.

The state system has access to most Salt interfaces. This means that from within a state, you are free to access the pillar, the grains and other Salt features. Additionally, the state system itself can be used by other Salt features as well. Salt reactors and runners for instance can also run states. Obviously, it is also possible to run a state from the CLI at will or to schedule states at regular intervals.

Another nice thing about the state system is that it allows you to use requisites to create relationships between different states and/or actions inside a single state. An example of what this would enable you to do is to make sure an action is executed only if another action was completed successfully.  

Though other options exist, most Salt states are written in YAML and Jinja. This makes a state easy to read, turning it into a nice abstraction layer to manage complexity.

The general philosophy is that a state should be idempotent. Regardless of how many times you run a state, it should bring a system into the same state, always.


### Execution modules

If you like Python, the (custom)-execution module will make you fall in love with Salt. The execution module is a Python script with functions that you can call on the command line or use inside a Salt state. 

{:refdef: style="text-align: center;"}
![Salt execution module ](/img/salt_execution_module.png "Salt execution module ")
{: refdef}

Salt comes with several execution modules 'out of the box'. Those modules can be found <a href="https://github.com/saltstack/salt/tree/master/salt/modules" target="_blank">here</a>. This is also a nice folder to browse in case you are looking to get started writing your own execution module. 

Writing your own is something I really recommend. An execution module you create yourself is referred to as a 'custom execution module'. Some of the things I really like about the custom execution module are the following:
- You can express yourself in Python.
- You can pip install whatever you want and use that in the execution module (system running the minion has to let you).
- Many Salt interfaces, like the pillar and grains, are available in the execution module through special dunder methods.
- The execution module can be used by other Salt features as well. You can call the execution module in states, custom states and in other execution modules.
- The master can utilize the message bus to have all minions run the execution module in (near) parallel.


### Custom states

A custom state is a construct that Salt offers you to write state functions in. You will be able to call these functions directly in the state system. The custom state is written in Python and it can interface with other Salt features. Some of the things you can do inside a custom state functions are:
- interface with pillar or grains data
- call execution modules or custom execution modules

The idea is that the execution module is a low level function that accomplishes a single task. The custom state is something that can be used to call different execution modules while putting in additional logic. In the end though, you are free to use it the way you want.

One thing to keep in mind is that unlike (custom-) execution modules, custom state functions cannot be called directly from the CLI. 


### Runners

Salt runners are similar to execution modules. The key difference between an execution module and a runner is the fact that a runner is executed on the salt-master instead of on the salt-minion. Because of this, in a salt-runner, the Python you write will have access to all the available Salt interfaces from the master perspective. Since the runner function is executed on the master, there is also the possibility to have it perform actions on several minions. 

{:refdef: style="text-align: center;"}
![SaltStack runner](/img/salt_runner_example.png "SaltStack runner")
{: refdef}


### Orchestration

Orchestration states can be used to do what the name implies: orchestrate. 

{:refdef: style="text-align: center;"}
![SaltStack orchestration state](/img/salt_orchestration_state.png "SaltStack orchestration state")
{: refdef}


Orchestration states are a collection of actions that are executed on the master. Some of actions that you can make the master perform are the following:
- instruct minions to run execution modules or states
- make the master execute a salt-runner
- use salt-ssh to perform a task on a remote system
- call additional orchestration states

### SalStack reactors and beacons

Reactors and beacons are the constructs that can be leveraged to make the master respond to events generated by the minion.

{:refdef: style="text-align: center;"}
![SaltStack reactors and beacons](/img/salt_reactor_and_beacon.png "SaltStack reactors and beacons")
{: refdef}

**Beacon**: Salt beacon is a minion-side feature that can be used to generate events. The events generated by a beacon are send to the master using the message bus. A beacon can be used for anything on the system that the minion controls or has access to. For instance, a beacon can be used to generate an event when a file is changed or when a network interface goes down. You could then use a reactor to determine if and what actions should follow in response to the beacon. 

**Reactor**: The reactor is a master-side interface that is used to watch the event bus for messages and respond to them. A reactor can be made to respond to a pattern, or tag that may or may not be generated by a beacon. The response can be to start a state or a runner.


### SaltStack Enterprise (SSE)

So far, everything that was discussed were the things that you find in <b>Salt open</b>. In addition to Salt open, there is also SSE. This is a commercial product that adds the following features to SaltStack:
- Centralized control of your salt masters through a GUI or an Enterprise API
- Role based access control
- Multi-master support
- LDAP and Active Directory integration
- Reporting
- Central job and event cache


## Closing thoughts

Working with Salt has been a lot of fun. The first weeks trying out different things using minions and proxy-minions, I struggled a lot. There were so many new concepts that I suddenly had to understand and work with. I hope this write-up will help others starting out with Salt understand it a little bit better.
