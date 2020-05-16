---
layout: post
title: Working with YAML in Python
tags: [ python, automation ]
image: /img/python-logo.jpg
---

### Python list comprehensions

Using Python list comprehensions allows you to use all of the things shown previously while writing it down in a very concise way. See the [Python intro](https://docs.python.org/3/tutorial/datastructures.html#list-comprehensions) for the basics on list comprehensions. 

The syntax for a list comprehension is the following:

{:refdef: style="text-align: center;"}
![Python list comprehension](/img/list_comprehension.png "Python list comprehension")
{: refdef}

In the following example, we will use a list comprehension to turn a string into a list of lines. We will do so without putting the empty line into the list:

```python
s = """
SoMe StRinG.   

Empty lInEs, leAding anD Trailing whiTespaces.   
  MiXed UPPER and lower case.
"""
new_list = [ line for line in s.splitlines() if line ]
```

The `new_list` now contains the following:

<pre style="font-size:12px">
['SoMe StRinG.   ', 'Empty lInEs, leAding anD Trailing whiTespaces.   ', '  MiXed UPPER and lower case.']
</pre>

The string was converted into a list. By putting in the <b>if line</b> test, we ensure that non-empty lines were not added to the list ( if '' evaluates to False).

Let's work on the strings in the list and remove unwanted whitespaces as well as make everything lowercase. To do this, we need to work on the expression part of the list comprehension:

{:refdef: style="text-align: center;"}
![Python list comprehension](/img/list_comprehension_new_string.png "Python list comprehension")
{: refdef}

The list comprehension now looks like this:

```python
new_list = [ line.lower().strip() for line in s.splitlines() if line ]
```

The <b>new_list</b> now contains the following items:

<pre style="font-size:12px">
['some string.', 'empty lines, leading and trailing whitespaces.', 'mixed upper and lower case.']
</pre>


Using <a href="https://docs.python.org/3/library/stdtypes.html?highlight=join#str.join" target="_blank">join</a> on the newly created list, we turn it into a new string. And instead of referencing the new list, we can just feed the list comprehension to <b>join</b>:

```python
new_string = '\n'.join([ line.lower().strip() for line in s.splitlines() if line ] )
print(new_string)
```

This will output the following:

<pre style="font-size:12px">
some string.
empty lines, leading and trailing whitespaces.
mixed upper and lower case.
</pre>



Now let's look at an example where we use a list comprehension to grab the dynamically learned MAC addresses on a switch:

```python
s = """
Vlan    Mac Address       Type        Ports      Moves   Last Move
----    -----------       ----        -----      -----   ---------
  13    0000.0c9f.f001    DYNAMIC     Po999      1       94 days, 3:54:30 ago
  13    00de.fbb9.cd41    DYNAMIC     Po992      1       94 days, 3:54:30 ago
  13    00de.fbba.aec1    DYNAMIC     Po999      1       94 days, 3:54:30 ago
  13    00de.fbba.avc1    DYNAMIC     Po996      1       94 days, 3:54:30 ago
  13    00de.fbda.a2c1    DYNAMIC     Po999      1       94 days, 3:54:30 ago
  13    00de.fb3a.avc1    DYNAMIC     Po996      1       94 days, 3:54:30 ago  
  13    7483.ef2c.bbb3    STATIC      Po1000
 870    0000.0c9f.f001    DYNAMIC     Po999      1       94 days, 3:54:30 ago
 870    00de.fbb9.cd41    DYNAMIC     Po999      1       94 days, 3:54:30 ago
"""

mac_list = [ mac.split()[1] for mac in s.splitlines()
            if 'DYNAMIC' in mac ]
```

In the previous example, we turn the string into a list of lines using <b>s.splitlines()</b>. Additionally, we performed <b>.split()[1]</b> on every line. This allowed us to extract the MAC address. After that, we checked for the presence of the word <b>DYNAMIC</b>. The `mac_list` will hold the following items after running the code above:
<pre style="font-size:12px">
['0000.0c9f.f001', '00de.fbb9.cd41', '00de.fbba.aec1', '00de.fbba.avc1', '00de.fbda.a2c1', '00de.fb3a.avc1', '0000.0c9f.f001', '00de.fbb9.cd41']
</pre>

If you want to narrow the MAC address list down to the MAC addresses in VLAN 13, you can add a condition to the list comprehension, like so:

```python
mac_list = [ mac.split()[1] for mac in s.splitlines()
            if 'DYNAMIC' in mac and
            mac.split()[0] == '13']
```

With that addition, the `mac_list` will hold the following items:

<pre style="font-size:12px">
['0000.0c9f.f001', '00de.fbb9.cd41', '00de.fbba.aec1', '00de.fbba.avc1', '00de.fbda.a2c1', '00de.fb3a.avc1']
</pre>

The configuration of some vendors can also be iterated using list comprehensions. Take a Juniper configuration retrieved using <b>show configuration | display set</b> for instance. The following would narrow the Juniper configuration down to lines that specify the interfaces in the routing-instances:

```python
vrf_interfaces = [ line for line in s.splitlines()
            if 'routing-instances' in line and
            'interface' in line ]
```

Similar is the case with an IOS-XR configuration that is outputted using <b>show running-config formal</b>.

```python
ospf_interfaces = [ line for line in s.splitlines()
            if 'router ospf' in line and
            'area 0' in line ]
```



Obviously, we can also use `any()` and `all()` inside list comprehensions:

```python
s = """
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-import-policy in
router bgp 65500 neighbor 10.0.19.1 address-family vpnv4 unicast route-policy bgp-export-policy out
set protocols bgp group exchange import exchange-import
set protocols bgp group exchange export exchange-export
set protocols bgp group exchange neighbor 10.0.0.1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 export deny-all
set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all
"""
interesting_items = [ 'exchange', 'import',  ]

interesting_lines = [ line for line in s.splitlines() if all(x in line for x in interesting_items) ]
```

This items in `interesting_lines` would be the following:
<pre style="font-size:12px">
['set protocols bgp group exchange import exchange-import', 'set protocols bgp group exchange neighbor 2001:DB8::1 import deny-all']
</pre>

Using <b>any</b> and <b>all</b> to test for multiple conditions looks a lot better than lengthy if statements. I'll leave it to you to determine whether or not they should be used in list comprehensions.