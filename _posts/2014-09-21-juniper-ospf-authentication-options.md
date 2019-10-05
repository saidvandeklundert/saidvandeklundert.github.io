---
layout: post
title: Juniper OSPF authentication options.
image: /img/juniper_logo.jpg
---
                   
<p>
    In Junos , OSPF authentication can come in one of three ways; none, simple or MD5.
</p>                    
<p>
    The default is to have no authentication. This means that the router will form a neighbor relationship with a neighboring router as long as the proper fields in the OSPF Hello’s are matching.
</p>
<p>
    Another option is to have simple (plain-text) authentication. When simple authentication is configured, every packet will include a plain-text password. 
</p>
<p>
    The third option offered by Junos is MD5 authentication. When this type of authentication is enabled, the OSPF packets will exchange an MD5 checksum. 
</p>
<p>
    When you want to use MD5 authentication on Junos, you have to configure a key under the interface configuration in OSPF. 
    The key consists of a number (anything between 0 and 255) and a password. 
    A thing worth noting is it that if multiple keys are configured, the key with the highest value will be used.
</p>
<p>
    Optionally, you can also define a start-time to indicate when the router should start using the key. 
    When you associate a start-time with a certain key, and the start-time is reached, that key will be used regardless of the values associated with the other keys. 
    When you have multiple keys with start-times configured, the key with the most recent start time will be selected. 
</p>
<p>
    This enables a smooth transition from old to new keys. Do make sure that the key values on both routers match. The key ID itself is exchanged as well and must match.
</p>
<p>
    That being said, let’s apply some authentication to the routers currently in the lab.
</p>

![OSPF authentication](/img/junos_ospf_authentication.png "OSPF authentication") 

<p>
    <b>
        Plain-text authentication.
    </b> 
    <br>
    <br>
    Configuring plain-text authentication can hardly be called a task:
</p>                    

![OSPF authentication](/img/juniper_jncip_sp_lab_1_episode_3_2.JPG "OSPF authentication") 

<p>
    The command above will have R13 start sending the plain-text password in every OSPF packet. 
    R13 will, from now on, also require that the neighboring router includes the same plain-text password in every packet.
</p>
<p>
    In order to see just how plain-text this plain-text is, let’s monitor traffic sent to the control plane of the router:
</p>

![OSPF authentication](/img/juniper_jncip_sp_lab_1_episode_3_3.JPG "OSPF authentication") 

<p>
    When we monitoring the interface, we can verify that all packets will include that plain-text password.
    The output below shows R12 sending an LSR to R13, during the loading phase of the neighbor relationship establishment:
</p>

![OSPF authentication](/img/juniper_jncip_sp_lab_1_episode_3_4.JPG "OSPF authentication") 

<p>
    <b>
        MD5 authentication.
    </b>
    <br>
    <br>
    Configuring MD5 authentication can be done in the following way;
</p>                    

![OSPF authentication](/img/juniper_jncip_sp_lab_1_episode_3_5.JPG "OSPF authentication") 

<p>
    After configuring R11 for MD5 authentication, the following can be seen when the interface is monitored:
</p>                    


![OSPF authentication](/img/juniper_jncip_sp_lab_1_episode_3_6.JPG "OSPF authentication") 

<p>
    Here we see that instead of the plain-text password, the key-id and a cryptographic sequence number is send together with each packet. 
    <br>
    Again, the key-id must match on both routers and the highest key that is configured will win.
</p>
<p>
    21-9-2014
</p>