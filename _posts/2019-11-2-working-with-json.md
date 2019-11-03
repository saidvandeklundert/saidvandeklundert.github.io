---
layout: post
title: Working with JSON in Python
tags: [automation, python]
image: /img/python-logo.png
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


Basic operations using JSON in Python
=====================================

There are multiple options available, but the only module I ever used is the one that is found in the Python Standard Library simply called [json](https://docs.python.org/3/library/json.html). The following Python 3.6 examples all use this library. 


<h2>Store Python dictionary as JSON in a file using dump</h2>

```python
#!/usr/bin/python3
from json import dump

d = {
    'Automate the Boring Stuff with Python' : 'Al Sweigart',
    'Fluent Python' : 'Luciano Ramalho',
    'Learning Python' : 'Mark Lutz',
    'Python Tricks' : 'Dan Bader',
}

with open('dictionary.json', 'w') as f:
    dump(d, f)
```


<h2>Load JSON as a Python dictionary from a file using load</h2>

```python
#!/usr/bin/python3
from json import load

with open('dictionary.json', 'r') as f:    
    d = load(f)
```

<h2>Emit JSON as string using dumps</h2>

```python
#!/usr/bin/python3
from json import load, dumps

with open('dictionary.json', 'r') as f:    
    d = load(f)

s = dumps(d)

print(s)
```

<h2>JSON kwargs to make the output prettier on the eyes</h2>

Works for <b>dump</b> as well as <b>dumps</b>:

```python
#!/usr/bin/python3
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
print(dumps(d, indent=4, separators=(',', ': ')))
```

<pre style="font-size:12px">
[said@test]$ <b>./pretty_json.py</b>
{"JSON is": ["case-sensitive", "does not care about whitespaces", "does not offer a way to put in comments", "valid YAML"]}
{
    "JSON is": [
        "case-sensitive",
        "does not care about whitespaces",
        "does not offer a way to put in comments",
        "valid YAML"
    ]
}
</pre>

<h2>JSON kwargs to sort the output:</h2>

Works for <b>dump</b> as well as <b>dumps</b>:

```python
#!/usr/bin/python3
from json import dumps

d = {"string": "word", "dict": {"a": "a", "b": "b"}, "list": [0, 1, 2], }
# Print the original:
print(dumps(d))
# Sort it by keys and print it:
print(dumps(d, sort_keys= True))
```

<h2>Dictify a web page</h2>

```python
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
```

<h2>Using JSON in jinja</h2>

```python
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
```


<h2>Conversion table for translating Python to JSON:</h2>


| Python data structures | serializes to JSON |
| ---------------------- | ------------------ |
| dict                   | object             |
| list, tuple            | array              |
| str                    | string             |
| int, float             | number             |
| True                   | true               |
| False                  | false              |
| None                   | null               |

Conversion table fortranslating JSON to Python:

| JSON value         | deserializes to Python |
| ------------------ | ---------------------- |
| object             | dict                   |
| array              | list                   |
| string             | str                    |
| number             | int, float             |
| true               | True                   |
| false              | False                  |
| null               | None                   |

https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/workgin_with_json/_posts/2019-11-2-working-with-json.md