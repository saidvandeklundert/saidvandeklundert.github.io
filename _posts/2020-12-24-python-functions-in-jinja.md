---
layout: post
title: Using Python functions in Jinja templates
tags: [ python, automation, ]
image: /img/python-logo.jpg
---

In some cases, Jinja templates can become too complicated. Lots of deeply nested if statements, clunky ways of working with variables and many more things that hurt the eyes.

{:refdef: style="text-align: center;"}
![Jinja logo](/img/jinja_logo.png "Jinja logo")
{: refdef}


What might be worth knowing is the fact that you can pass a Python function into your Jinja templates. Doing this can greatly improve the readability of your template as well as allow you to handle more complicated scenario's.


# Using a Python function in a Jinja template

The following Python will pass 2 functions to the Jinja template.

First, 2 functions are defined: 'hello_world' and 'multiply'. These functions are placed in a dictionary.

After this, the <b>render</b> function is created. Inside this function, we use <b>jinja_template.globals.update(func_dict)</b> to pass the previously created <b>func_dict</b> and expose it during the Jinja rendering phase. 


<pre style="font-size:12px">
# !/usr/bin/env python3
from jinja2 import Environment, FileSystemLoader


def hello_world():
    return "hello world from within the function"


def multiply(x, y):
    return str(x * y)


func_dict = {
    "hello_world": hello_world,
    "multiply": multiply,
}

def render(template=None):
    env = Environment(loader=FileSystemLoader("/var/tmp/"))
    jinja_template = env.get_template(template)
    jinja_template.globals.update(func_dict)
    template_string = jinja_template.render()
    return template_string


if __name__ == "__main__":
    print(render(template="test.j2"))
</pre>

In the following example Jinja, we use the functions like so:

<pre style="font-size:12px">
Calling the 'hello_world' function:
{{ hello_world() }}

Calling the 'multiply' function:
{{ multiply(2, 2) }}
</pre>

When we run the Python script, we get the following result:

<pre style="font-size:12px">
Calling the 'hello_world' function:
hello world from within the function

Calling the 'multiply' function:
4
</pre>


Being able to use Python functions like this inside a Jinja template has been very useful to me. It increased the readability of some templates as I was able to replace large parts with a straightforward Python functions. 

It also allowed me to handle more complex logic in a Python function. This allowed me to introduce more complicated configurations as well make templates behave based on the state of other systems. 

It is not something that is required all the time, but it is nice to know that it is a possibility.

