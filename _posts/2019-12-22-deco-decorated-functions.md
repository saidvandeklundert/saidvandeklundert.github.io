---
layout: post
title: parallelize Python programs using deco
tags: [automation, python]
image: /img/python-logo.jpg
---

During my first struggles with threading and multiprocessing, a colleague told me about <b>deco</b>. This package enables you to parallelize a simple function in a very easy way, making it run significantly faster. The package author Alex Sherman puts it like this:

<b>A simplified parallel computing model for Python. DECO automatically parallelizes Python programs, and requires minimal modifications to existing serial programs.</b>

I was blown away at how easy it was to use <b>deco</b>. To demonstrate this, let's use <b>deco</b> to parallelize the following script.


Slow ping
=========

<pre style="font-size:12px">
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


host_list = [ 'server-{}'.format(nr) for nr in range(0, 10) ]

for host in host_list:
    print(ping(host))
</pre>

The previous example runs through a list and pings every host in it. In an environment where every server was reachale, I got the following output after running the script:

<pre style="font-size:12px">
<b>sh-4.4# time python3 ping.py</b>
True
..
True
0:00:10.631077
</pre>   

The script took 10 seconds to complete. 


Making it faster with deco
==========================

Using <b>deco</b>, there are two things we need to do in order to speed up our previous example script. 

First we decorate the `ping()` function with `@concurrent`. After this, we create a function that executes the `ping()` function for every host in a list. In that function, we collect the results of the now decorated `ping()` function. All the results of the `ping()` function are returned as a list.

The script now looks like this:

<pre style="font-size:12px">
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

host_list = [ 'server-{}'.format(nr) for nr in range(0, 10) ]

print(ping_list(host_list))
</pre>

The `ping()` function was decorated with `concurrent`. In addition to this, there is a new function called `ping_list()`. In this function, there is the following list comprehension:

<pre style="font-size:12px">
ping_returns = [ ping_return.get()[0] for ping_return in map(ping, host_list)]    
</pre>

The `map(ping, host_list)` part basically runs `ping()` for every item in the `host_list`.

The other part of the list comprehension, `ping_return.get()[0] for ping_return`, collects the results and extracts from the result the part we are interested in. The `.get()[0]` is required because of the way that the deco-decorated function returns it's result.

When we run it now, we get the following:

<pre style="font-size:12px">
<b>sh-4.4# time python3 ping.py</b>
[True, True, True, True, True, True, True, True, True, True]
0:00:01.359208
</pre>

Only 1.3 seconds as opposed to 10.

The difference becomes more apparent as we expand the `host_list` to more servers like so:

<pre style="font-size:12px">
host_list = [ 'server-{}'.format(nr) for nr in range(0, 200) ]
</pre>

After expanding the list to include 200 severs, it took the script without <b>deco</b> about 3 minutes and 30 seconds to complete. The deco-decorated script only took about 4 seconds!


Closing thoughts
================

Using <b>deco</b> is very easy. I have used it in the past in circumstances where I needed a script to quickly send SNMP requests or some ICMPs to devices. 

The beauty of using <b>deco</b> is that the implementation requires so little effort. You can read more about <b>deco</b> [here](https://github.com/alex-sherman/deco). 
