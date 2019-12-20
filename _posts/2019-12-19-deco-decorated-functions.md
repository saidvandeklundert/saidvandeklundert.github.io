---
layout: post
title: Using deco-decorate functions in Python
tags: [automation, python]
image: /img/python-logo.jpg
---

During my first struggles with threading and multiprocessing, a collegue told me about `deco`. This package enables you to parallelize a function in a more simpe way, making it run significantly faster. The package author Alex Sherman puts it like this:

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
    Sends 3 ICMPs to a host and supress the output by sending it to devnull.
    
    Arg:
        dev: an object that is capable of communicating with the Juniper API.
    
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
    'google.com',
    'nu.nl',
    'facebook.com',
    '8.8.8.8',
    '1.1.1.1',
    '8.8.4.4',
    '9.9.9.9',
    'ad.nl',
    'saidvandeklundert.nl',
    'google.nl',      
]

for host in host_list:
    print(check_ping(host))

print(datetime.now() - startTime)
```

When we run this script, we get the following output:

<pre style="font-size:12px">
<b>sh-4.4# python3 /var/tmp/test_1.py</b>
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
    Sends 3 ICMPs to a host and supress the output by sending it to devnull.
    
    Arg:
        dev: an object that is capable of communicating with the Juniper API.
    
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
    'google.com',
    'nu.nl',
    'facebook.com',
    '8.8.8.8',
    '1.1.1.1',
    '8.8.4.4',
    '9.9.9.9',
    'ad.nl',
    'saidvandeklundert.nl',
    'google.nl',      
]

print(ping_list(host_list))

print(datetime.now() - startTime)
```

When we run it now, we get the following:

<pre style="font-size:12px">
<b>sh-4.4# python3 /var/tmp/test_2.py</b>
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











====


From `DECO: Polishing Python Parallel Programming`:
```
In the general case, adapting a program for use

with DECO would mean finding a worker func-
tion that is being called in the body of a loop

and is working on independent sections of data

in each call.
```





