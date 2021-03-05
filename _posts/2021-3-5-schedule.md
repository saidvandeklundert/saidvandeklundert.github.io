---
layout: post
title: Scheduling jobs in Python using schedule
tags: [ python, automation ]
image: /img/python-logo.jpg
---


The <b>schedule</b> module describes itself as 'Python job scheduling for humans'. It is a nice package that I use for infrastructure related tasks and activities. The package (currently) prides itself for the following:
- A simple to use API for scheduling jobs.
- Very lightweight and no external dependencies.
- Excellent test coverage.
- Tested on Python 3.6, 3.7, 3.8, 3.9

Here is an example on how schedule could be used to run a job that collects the backup for a collection of devices:

<pre style="font-size:12px">
import schedule
import time
from datetime import datetime


def collect_device_backup():
    now = datetime.now().strftime("%H:%M:%S")
    print(f"Collecting device backup at {now}.")


def data_harvester():    
    schedule.every(10).seconds.do(collect_device_backup)
    while True:
        schedule.run_pending()
        time.sleep(1)

if __name__ == "__main__":
    data_harvester()
</pre>

When we run this code, the backup job will run at a 10 second interval:

<pre style="font-size:12px">
Collecting device backup at 11:28:28.
Collecting device backup at 11:28:38.
Collecting device backup at 11:28:49.
Collecting device backup at 11:28:59.
</pre>

Note that with <b>schedule.run_pending()</b>, the intended behaviour for that method is to run jobs that are pending. Something scheduled to run every 10 seconds can only be run often enough if <b>run_pending()</b> is called at the same interval or faster. For instance, a job with a 10 second interval will only run once per minute if <b>run_pending()</b> is called once per minute. 

Let's proceed and add another job to the schedule. Instead of just collecting the backup, we also collect additional facts that describe the device:

<pre style="font-size:12px">
import schedule
import time
from datetime import datetime

def collect_device_backup():
    now = datetime.now().strftime("%H:%M:%S")
    print(f"Collecting device backup at {now}.")


def collect_device_facts():
    now = datetime.now().strftime("%H:%M:%S")
    print(f"Collecting device facts at {now}.")    
 

def data_harvester():    
    schedule.every(10).seconds.do(collect_device_backup)
    schedule.every(10).seconds.do(collect_device_facts)
    while True:
        schedule.run_pending()
        time.sleep(1)

if __name__ == "__main__":
    data_harvester()
</pre>

We are now running 2 tasks at the same interval:

<pre style="font-size:12px">
Collecting device backup at 11:35:06.
Collecting device facts at 11:35:06.
Collecting device backup at 11:35:16.
Collecting device facts at 11:35:16.
Collecting device backup at 11:35:26.
Collecting device facts at 11:35:26.
</pre>

If we keep adding tasks to run at the same interval, it might be wise to ensure that not everything is run at the same time. The <b>schedule</b> module offers this facility for us. Instead of using <b>every(10).seconds</b> we use <b>every(10).to(20).seconds</b>:

<pre style="font-size:12px">
def data_harvester():    
    schedule.every(10).to(20).seconds.do(collect_device_backup)
    schedule.every(10).to(20).seconds.do(collect_device_facts)
    while True:
        schedule.run_pending()
        time.sleep(1)
</pre>

After the fist run, schedule will use <b>random.randint</b> to randomize the time at which the tasks are executed. With this in place, we now see the following:

<pre style="font-size:12px">
Collecting device facts at 11:39:31.
Collecting device backup at 11:39:32.
Collecting device facts at 11:39:43.
Collecting device backup at 11:39:53.
Collecting device facts at 11:40:02.
Collecting device backup at 11:40:10.
Collecting device facts at 11:40:13.
</pre>

Notice the <b>do()</b> method ('<b>do</b>(collect_device_backup)'). It uses [partial](https://docs.python.org/3/library/functools.html#functools.partial) from <b>functools</b> to run the scheduled job. The <b>do()</b> method will pass additional <b>args</b> and <b>kwargs</b> to the function that is passed to <b>partial</b>.

In the next example, the <b>args_and_kwargs()</b> function is scheduled. It is passed 2 arguments and 2 keyword arguments. We do this using <b>do(args_and_kwargs, 'arg1', 'arg2', keyword1='argument1', keyword2='argument2')</b>:

<pre style="font-size:12px">
import schedule
import time

def args_and_kwargs(*args, **kwargs):
    print(f"Args:{args}.")
    print(f"kwargs:{kwargs}.")

 
def data_harvester():    
    schedule.every(10).seconds.do(args_and_kwargs, 'arg1', 'arg2', keyword1='argument1', keyword2='argument2')
    while True:
        schedule.run_pending()
        time.sleep(1)

if __name__ == "__main__":
    data_harvester()
</pre>

Running this will give you the following:

<pre style="font-size:12px">
Args:('arg1', 'arg2').
kwargs:{'keyword1': 'argument1', 'keyword2': 'argument2'}.
Args:('arg1', 'arg2').
kwargs:{'keyword1': 'argument1', 'keyword2': 'argument2'}.
Args:('arg1', 'arg2').
kwargs:{'keyword1': 'argument1', 'keyword2': 'argument2'}.
Args:('arg1', 'arg2').
kwargs:{'keyword1': 'argument1', 'keyword2': 'argument2'}.
</pre>

The schedule package offers more convenient ways to schedule jobs. The following comes from directly from the readme:

<pre style="font-size:12px">
import schedule
import time

def job():
    print("I'm working...")

schedule.every(10).seconds.do(job)
schedule.every(10).minutes.do(job)
schedule.every().hour.do(job)
schedule.every().day.at("10:30").do(job)
schedule.every(5).to(10).minutes.do(job)
schedule.every().monday.do(job)
schedule.every().wednesday.at("13:15").do(job)
schedule.every().minute.at(":17").do(job)

while True:
    schedule.run_pending()
    time.sleep(1)
</pre>

Check the package right [here](https://github.com/dbader/schedule/tree/master/schedule) to see what it has to offer.

For these examples, I used Python `3.9.0` and schedule version `1.0.0`.

