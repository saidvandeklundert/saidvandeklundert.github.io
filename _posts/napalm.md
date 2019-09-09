---
layout: post
title: NAPALM
image: /img/napalm_logo.png.png
---

Recently, I have become a bit more interested in NAPALM. Up untill this point, most of my scripting efforts involved PyEZ, paramiko and Netmiko. After having to work with multiple vendors and SaltStack, NAPALM started to come up more and more. This post is a short summary of what I have learned so far about NAPALM in general.  

![NAPALM](/img/napalm_logo.png "NAPALM")

<br>

NAPALM in a nutshell
====================

Different networking vendors ship their boxes/software with their own specific OS and every OS has its own interfaces. By now, most vendors have ensured their software ships with an API. In case you have a multivendor network, one of the challenges you will run into when you want to start automating, is figuring out how all these different APIs work.

Wouldn’t it be nice if there was some sort of library you could import and be done with it? Like a wrapper around all the vendor-specific things.

Enter NAPALM.

Though it does not abstract everything away, it is quite convenient for a lot of basic operations.

<br>

NAPALM, Network Automation and Programmability Abstraction Layer with Multivendor support
=========================================================================================


At the time of writing, the docs ( https://napalm.readthedocs.io/en/latest/ ) indicate the NAPALM offers support for the following devices:
- Arista EOS
- Cisco IOS
- Cisco IOS-XR
- Cisco NX-OS
- Juniper JunOS

These 'core' drivers are supported and maintained by the people working on NAPALM. In addition to the core drivers, there are also some 'community drivers'. These are maintained under their own repository and can be used by NAPALM.

The core drivers in NAPALM allow it to deal with the different devices. These drivers are libraries written to deal with the different vendor APIs. The following picture shows the current core libraries:

  =====================   ==========  =============   ============ ============  ============ ============
                          EOS         Junos           IOS-XR       NX-OS         NX-OS SSH    IOS
  =====================   ==========  =============   ============ ============  ============ ============
  **Driver Name**         eos         junos           iosxr        nxos          nxos_ssh     ios
  **Backend library**     `pyeapi`_   `junos-eznc`_   `pyIOSXR`_   `pynxos`_     `netmiko`_   `netmiko`_
  =====================   ==========  =============   ============ ============  ============ ============

.. _pyeapi: https://github.com/arista-eosplus/pyeapi
.. _junos-eznc: https://github.com/Juniper/py-junos-eznc
.. _pyIOSXR: https://github.com/fooelisa/pyiosxr
.. _pynxos: https://github.com/networktocode/pynxos
.. _netmiko: https://github.com/ktbyers/netmiko

As explained earlier, these backend libraries are used by NAPALM to communicate with the different vendors. The two backend libraries that stood out to me were `junos-eznc` and `netmiko`. The first one is familiar through earlier work with PyEZ. The latter was familiar for some scripting I did against several other vendors. The `netmiko` library, when used outside of NAPALM as standalone library, is a `Multi-vendor library to simplify Paramiko SSH connections to network devices`. To enable NAPALM to work with NX-OS over SSH and IOS (which does not have an API), they opted to use this as a backend library.

<br>

NAPALM in a Python script
=========================

So when you use NAPALM in a Python script, what exactly happens? Since I am most familiar with Juniper, I thought I'd explain it from a Juniper point of view:

![NAPALM talking to Junos](/img/napalm_talking_to_junos.png "NAPALM talking to Junos")

At the top is your Python script, this is where you import NAPALM. When you talk to a Juniper device, you tell NAPALM to use the `junos` driver, which will in turn reference to the PyEZ backend library (`junos-eznc`).This library uses NETCONF to communicate with the Juniper XML API. This API is an interface of `MGD` that exposes `Junos` and allows you to pass instructions or retreive information.

<br>

NAPALM outside of a Python script
=================================


As it says on the NAPALM website (https://napalm-automation.net/ ), `Napalm plays nice with others!`. Efforts have been made to enable the use of NAPALM outside of a Python script. You can use it for Ansible, Stackstorm and SaltStack. In the case of SaltStack, the automation framework I am most familiar with, this was done through the creation of a proxy minion and several modules that exposes most of the methods available in NAPALM.











Supported devices:
https://napalm.readthedocs.io/en/latest/support/index.html

IOS caveats:
https://napalm.readthedocs.io/en/latest/support/ios.html




In this post,

main link: https://github.com/napalm-automation/napalm





NAPALM:
- Napalm is a vendor neutral, cross-platform open source project that provides a unified API to network devices.
- Napalm plays nice with others! Ansible, SaltStack, StackStorm


IOS-XR uses pyIOSXR:
https://github.com/napalm-automation/napalm/blob/develop/napalm/iosxr/iosxr.py

Junos uses PyEZ:
https://github.com/napalm-automation/napalm/blob/develop/napalm/junos/junos.py

EOS uses pyeapi:
https://github.com/napalm-automation/napalm/blob/develop/napalm/eos/eos.py


Closing
============

my connection
- SaltStack
- IOS XR
- Junos
- Arista