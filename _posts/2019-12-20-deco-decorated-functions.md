---
layout: post
title: Using deco-decorate functions in Python
tags: [automation, python]
image: /img/python-logo.jpg
---

During my first struggles with threading and multiprocessing, a colleaugue told me about `deco`. This package enables you to parallelize a function in a very easy way, making it run significantly faster. The package author Alex Sherman puts it like this:

`A simplified parallel computing model for Python. DECO automatically parallelizes Python programs, and requires minimal modifications to existing serial programs.`

I was blown away at how easy it was to use `deco`. To demonstrate how easy it is, let's look at 2 example scripts. First, a script that runs through a list and sends a ping to every host in that list. After this, we will use `deco` to demonstrate how easy we can parallelize the first script.


Slow ping
=========

Have a look at the following example script:

```python
import os
import subprocess

def ping(host):
    '''
    Sends 2 ICMPs to a host and suppress the output by sending it to devnull.
    
    Returns True if there was a response, False otherwise.
    '''
    with open(os.devnull, 'w') as devnull:
        result = subprocess.call(
            ['ping', '-c', '2', '-W', '2', host],
            stdout=devnull,
            stderr=devnull
            )
            
    if result == 0:
        return True
    else:        
        return False


host_list = [
    'host-1',
    'host-2',
    'host-3',
    'host-4',
    'host-5',
    'host-6',
    'host-7',
    'host-8',
    'host-9',   
]

for host in host_list:
    print(ping(host))
```

When we run this script, we get the following output:

<pre style="font-size:12px">
<b>sh-4.4# time python3 ping.py</b>
True
True
True
True
True
True
True
True
True
True
0:00:10.631077
</pre>   

We iterated a list and executed `ping()` for every host in the list. It took us 10 seconds. If I expand the list to 200 hosts, it takes about 3 minutes and 30 seconds.


Making it faster with deco
==========================

Using `deco`, there are two things we need to do in order to speed up our previous example script. 

First we decorate the `ping()` function with `@concurrent`. After this, we create a function that executes the `ping()` function for every host in a list. In that function, we collect the results of `ping()` and return them as a list.

The script now looks like this:

```python
import os
import subprocess
from deco import concurrent, synchronized

@concurrent
def ping(host):
    '''
    Sends 2 ICMPs to a host and suppress the output by sending it to devnull.
    
    Returns True if there was a response, False otherwise.
    '''
    with open(os.devnull, 'w') as devnull:
        result = subprocess.call(
            ['ping', '-c', '2', '-W', '2', host],
            stdout=devnull,
            stderr=devnull
            )
            
    if result == 0:
        return True
    else:        
        return False


@synchronized
def ping_list(host_list):
    """
    Runs ping() against a list of hosts in parallel and returns the
    results as a list.
    """
    ping_returns = [ ping_return.get()[0] for ping_return in map(ping, host_list)]    
        
    return ping_returns

host_list = [
    'host-0',    
    'host-1',
    'host-2',
    'host-3',
    'host-4',
    'host-5',
    'host-6',
    'host-7',
    'host-8',
    'host-9',
]

print(ping_list(host_list))
```

The `ping()` function was decorated with `concurrent`. In addition to this, there is a new function called `ping_list()`. In this function, there is the following list comprehension:

```python
ping_returns = [ ping_return.get()[0] for ping_return in map(ping, host_list)]    
```

Let's start with the `map(ping, host_list)` part. This basically runs `ping()` for every item in the `host_list`.

The other part of the list comprehension, `ping_return.get()[0] for ping_return`, collects the results and extracts from the result the part we are interested in. The `.get()[0]` is required because of the way that the deco-decorated function returns it's result.

When we run it now, we get the following:

<pre style="font-size:12px">
<b>sh-4.4# time python3 ping.py</b>
[True, True, True, True, True, True, True, True, True, True]
0:00:01.359208
</pre>

Only 1.3 seconds as opposed to 10.

And if I expand the list to 200 hosts, it only takes about 4 seconds!


Closing thoughts
================

Using `deco` is very easy. I have used it in the past in circumstances where I needed a script to quickly send SNMP requests, pings, or some CLIs to devices. 

The beauty of using `deco` is that the implementation requires so little effort.

You can pip install the package and you can read more about it `deco` [here](https://github.com/alex-sherman/deco). 
