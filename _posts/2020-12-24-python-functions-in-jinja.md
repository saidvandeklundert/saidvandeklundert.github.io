---
layout: post
title: Using Python functions in Jinja templates
tags: [ python, automation, ]
image: /img/python-logo.jpg
---




Jinja is a templating engine for the Python programming language. It is not a programming language. Jinja comes with its own syntax, control structures, expressions and it allows you to do some pretty complicated things.

That does not mean it is a good idea.

{:refdef: style="text-align: center;"}
![Jinja logo](/img/jinja_logo.png "Jinja logo")
{: refdef}

In some cases, templates become too complicated. Especially when there is too much logic that needs to be dealt with inside the Jinja template. As a result, you can see a lot of deeply nested if statements, clunky ways of working with variables, filtering the variables and more. 

What might be worth knowing is the fact that you can pass a Python function into your Jinja templates. Doing this can greatly improve the readability of your template as well as allow you to handle more complicated scenario's.


# The Python script

The following is a Python script that will render a template using the <b>render</b> function. Inside that function, we use <b>jinja_template.globals.update(func_dict)</b> to pass in a dictionary called <b>func_dict</b>. This dictionary contains 2 functions: 'hello_world' and 'multiply'.


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

We can now use the functions inside the Jinja template like so:

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


Being able to use Python functions like this inside a Jinja template has been useful to me. It increased the readability of some templates as I was able to replace large parts with a straightforward Python functions. 

It also allowed me to handle more complex logic in a Python function. This allowed me to introduce more complicated configurations as well make templates behave based on the state of other systems. 

It is not something that is required all the time, but it is nice to know that it is a possibility.

