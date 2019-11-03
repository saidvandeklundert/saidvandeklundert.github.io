---
layout: post
title: Working with JSON in Python
tags: [automation, python]
image: /img/python-logo.jpg
---

JavaScript Object Notation, or JSON, is something that started popping up more and more the moment I started doing (Net)DevOps. These are some of my notes and examples.


JSON overview
=============

JSON is an open standard format that can be used to store or transmit data. The two data structures available in JSON are:
-	<b>Objects</b>: an unordered collection of one or more key-value pairs enclosed in braces <b>{}</b>.
-	<b>Arrays</b>: and ordered collection of values enclosed in brackets <b>[]</b>.

An object is similar to a Python dictionary and an array is similar to a Python list.

The values that you can use or come across in these data structures are the following:
-	String (0 or more Unicode characters)
-	Number (integer or float)
-	true
-	false
-	null
-	Object
-	Array


JSON basics in Python
=====================

There are multiple options available, but the only module I ever used is the one that is found in the Python Standard Library simply called [json](https://docs.python.org/3/library/json.html). The following Python 3.6 examples all use this library. 


<h2>Store Python dictionary as JSON in a file using dump</h2>

<pre style="font-size:12px">
#!/usr/bin/python3
from json import dump

d = {
    'Automate the Boring Stuff with Python' : 'Al Sweigart',
    'Fluent Python' : 'Luciano Ramalho',
    'Learning Python' : 'Mark Lutz',
    'Python Tricks' : 'Dan Bader',
}

with open('/var/tmp/dictionary.json', 'w') as f:
    dump(d, f)
</pre>

Note: any file opened using <b>with</b> is automatically closed when the script is done using the file.

<h2>Load JSON as a Python dictionary from a file using load</h2>

<pre style="font-size:12px">
#!/usr/bin/python3
from json import load

with open('/var/tmp/dictionary.json', 'r') as f:    
    d = load(f)
</pre>

<h2>Emit JSON as string using dumps</h2>

<pre style="font-size:12px">
#!/usr/bin/python3
from json import load, dumps

with open('/var/tmp/dictionary.json', 'r') as f:    
    d = load(f)

s = dumps(d)

print(s)
</pre>

<h2>JSON kwargs to make the output prettier on the eyes</h2>

Works for <b>dump</b> as well as <b>dumps</b>:

<pre style="font-size:12px">
{% raw %}#!/usr/bin/python3
from json import dumps

d = {
    "JSON is": [
        'case-sensitive',
        'does not care about whitespaces',
        'does not offer a way to put in comments',
        'valid YAML',
        ], 
    }
# Print the original:
print(dumps(d))
# Print the pretty output:
print(dumps(d, indent=4, separators=(',', ': '))){% endraw %}
</pre>

Output to show the difference:

<pre style="font-size:12px">
{% raw %}{"JSON is": ["case-sensitive", "does not care about whitespaces", "does not offer a way to put in comments", "valid YAML"]}

{
    "JSON is": [
        "case-sensitive",
        "does not care about whitespaces",
        "does not offer a way to put in comments",
        "valid YAML"
    ]
}{% endraw %}
</pre>

<h2>JSON kwargs to sort the output:</h2>

Works for <b>dump</b> as well as <b>dumps</b>:

<pre style="font-size:12px">
#!/usr/bin/python3
from json import dumps

d = {"string": "word", "dict": {"a": "a", "b": "b"}, "list": [0, 1, 2], }

# Sort it by keys and print it:
print(dumps(d, sort_keys= True))
</pre>

<h2>Dictify a web page</h2>

<pre style="font-size:12px">
#!/usr/bin/python3
import json
from pprint import pprint
from urllib.request import urlopen

# Open the web page:
html_content = urlopen('http://validate.jsontest.com/?json=%5BJSON-code-to-validate%5D').read()  

# Load the JSON:
d_json_test = json.loads(html_content)

# Print the JSON to screen:
pprint(d_json_test)

# All in one line:
pprint(json.loads(urlopen('http://validate.jsontest.com/?json=%5BJSON-code-to-validate%5D').read()))
</pre>

The last line of the script outputs to:

<pre style="font-size:12px">
{'empty': False,
 'object_or_array': 'array',
 'parse_time_nanoseconds': 29928,
 'size': 1,
 'validate': True}
</pre>

<h2>Using JSON in jinja</h2>

<pre style="font-size:12px">
#!/usr/bin/python3
import json
from jinja2 import Template

json_str = '''
    [
    {"hostname": "ny01", "mgmt-ip": "10.0.0.1" }, 
    {"hostname": "ny02", "mgmt-ip": "10.0.0.2" }, 
    {"hostname": "ny03", "mgmt-ip": "10.0.0.3" }, 
    {"hostname": "ny04", "mgmt-ip": "10.0.0.4" }
    ]
    '''
# Load json from string:
routers = json.loads(json_str)

# Define template:
template = Template('''

{% for router in routers %}
set system host-name {{ router['hostname'] }}
set interfaces lo0 unit 0 family inet address {{ router['mgmt-ip'] }} primary
{% endfor %}

''')

# Render template and send the output to screen:
print(template.render(routers = routers))
</pre>

Outputs the following:

<pre style="font-size:12px">
set system host-name ny01
set interfaces lo0 unit 0 family inet address 10.0.0.1 primary

set system host-name ny02
set interfaces lo0 unit 0 family inet address 10.0.0.2 primary

set system host-name ny03
set interfaces lo0 unit 0 family inet address 10.0.0.3 primary

set system host-name ny04
set interfaces lo0 unit 0 family inet address 10.0.0.4 primary
</pre>


Thing to keep in mind:
======================


When you are translating JSON to Python or vice versa, the following conversion tables apply:

<b>Conversion table for translating Python to JSON:</b>


| Python data structures | serializes to JSON |
| ---------------------- | ------------------ |
| dict                   | object             |
| list, tuple            | array              |
| str                    | string             |
| int, float             | number             |
| True                   | true               |
| False                  | false              |
| None                   | null               |

<b>Conversion table fortranslating JSON to Python</b>:

| JSON value         | deserializes to Python |
| ------------------ | ---------------------- |
| object             | dict                   |
| array              | list                   |
| string             | str                    |
| number             | int, float             |
| true               | True                   |
| false              | False                  |
| null               | None                   |

