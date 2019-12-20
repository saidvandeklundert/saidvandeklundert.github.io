---
layout: post
title: Using deco-decorate functions in Python
tags: [automation, python]
image: /img/python-logo.jpg
---

During my first struggles with threading and multiprocessing, a colleaugue told me about `deco`. This package enables you to parallelize a function in a very easy way, making it run significantly faster. The package author Alex Sherman puts it like this:

`A simplified parallel computing model for Python. DECO automatically parallelizes Python programs, and requires minimal modifications to existing serial programs.`

I was blown away at how easy it was to use `deco`. To demonstrate how easy it is, let's look at 2 example scripts. First, a script that runs through a list and sends a ping to every host in that list. After this, we will use `deco` to demonstrate how easy we can parallelize the first script.


Sending ICMPs one after the other
=================================

Have a look at the following example script:

```python
import os
import subprocess
from datetime import datetime
from deco import *

def check_ping(host):
    '''
    Sends 3 ICMPs to a host and suppress the output by sending it to devnull.
    
    Arg:
        host: an IP address or a hostname the system can resolve
    
    Returns:
        0 if there was a response
        1 if there was no response
    '''
    host = str(host)
    result = subprocess.call(['ping', '-c', '2', '-W', '2', host],
                           stdout=open(os.devnull, 'w'),
                           stderr=open(os.devnull, 'w'))
    if result == 0:
        return True
    else:        
        return False

startTime = datetime.now()
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

for host in host_list:
    print(check_ping(host))

print(datetime.now() - startTime)
```

When we run this script, we get the following output:

<pre style="font-size:12px">
<b>sh-4.4# python3 ping.py</b>
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

We iterated a list and executed `check_ping()` for every host in the list. It took us 10 seconds. If I expand the list to 200 hosts, it takes about 3 minutes and 30 seconds.


Using deco
==========

Using `deco`, we only need to implement some minor modifications to make the previous example script go a lot faster.

The summary of what `deco` does from the package author:
```
As an overview, DECO is mainly just a smart wrapper for Python's multiprocessing.pool. When @concurrent is applied to a function it replaces it with calls to pool.apply_async. Additionally when arguments are passed to pool.apply_async, DECO replaces any index mutable objects with proxies, allowing it to detect and synchronize mutations of these objects. The results of these calls can then be obtained by calling wait() on the concurrent function, invoking a synchronization event. These events can be placed automatically in your code by using the @synchronized decorator on functions that call @concurrent functions. Additionally while using @synchronized, you can directly assign the result of concurrent function calls to index mutable objects. These assignments get refactored by DECO to automatically occur during the next synchronization event. All of this means that in many cases, parallel programming using DECO appears exactly the same as simpler serial programming.
```

There are two things that we need to do in order to speed up our previous example script. First we decorate the `check_ping()` function with `@concurrent`. After this, we create a function that steps through a list and executes the `check_ping()` function. In that function, we can collect the results of `check_ping()` and return them in the way we want. In the following example, I chose to return it as a list.

The script now looks like this:

```python
import os
import subprocess
from datetime import datetime
from deco import *

@concurrent
def check_ping(host):
    '''
    Sends 3 ICMPs to a host and suppress the output by sending it to devnull.
    
    Arg:
        host: an IP address or a hostname the system can resolve
    
    Returns:
        0 if there was a response
        1 if there was no response
    '''
    host = str(host)
    result = subprocess.call(['ping', '-c', '2', '-W', '2', host],
                           stdout=open(os.devnull, 'w'),
                           stderr=open(os.devnull, 'w'))
    if result == 0:
        return True
    else:        
        return False


@synchronized
def ping_list(host_list):
    """
    Runs check_ping(host) against a list of hosts in parallel.

    Arg:
        host_list: a list of hosts
    
    Returns:
        A list containing the result for every single time the check_ping(host) ran
    
    """
    check_ping_results = [ x for x in map(check_ping, host_list)]

    results = [x.get()[0] for x in check_ping_results ]
    
    return results


startTime = datetime.now()
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

print(datetime.now() - startTime)
```

The `check_ping()` function was decorated with `concurrent`. In addition to this, there is a new function called `ping_list()`. In this function, there are 2 list comprehensions. 

The first list comprehension was the following:
```python
check_ping_results = [ x for x in map(check_ping, host_list)]
```

Here, you see `map()`, which will 'Return an iterator that applies function to every item of iterable'. Basically, it runs `check_ping()` for every item in the `host_list`. The list comprehension stores the returns in the `check_ping_results` list.

When we run the decorated function, the return is slightly modified. The second list comprehension deals with this:

```python
results = [x.get()[0] for x in check_ping_results ]
```

Here, we extract the `check_ping()` result we are after by doing `x.get()[0]` on every return we previously stored in the `check_ping_results`.

When we run it now, we get the following:

<pre style="font-size:12px">
<b>sh-4.4# python3 ping.py</b>
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