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

An object is similar to a Python dictionary and an array is similar to a list in Python.

The values that you will find in these data structures are the following:
-	String (0 or more Unicode characters)
-	Number (integer or float)
-	true
-	false
-	null
-	Object
-	Array


JSON is case-sensitive, does not care about whitespaces and does not offer a way to put in any comments. Another thing that is nice to know is that every JSON file is also a valid YAML file.

Conversion table for translating Python to JSON:

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


Basic operations using JSON in Python
=====================================

There are multiple options available, but the only module I ever used is the one that is found in the Python Standard Library simply called [json](https://docs.python.org/3/library/json.html). The following Python 3.6 examples all use this library.


<b>Store Python dictionary as JSON in a file</b>

```python
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

<b>Load JSON as a Python dictionary from a file</b>

```python
#!/usr/bin/python3
from json import load

with open('dictionary.json', 'r') as f:    
    d = load(f)
```

<b>Emit JSON as string</b>

```python
#!/usr/bin/python3
from json import load, dumps

with open('dictionary.json', 'r') as f:    
    d = load(f)

s = dumps(d)

print(s)
```

<b>JSON kwargs to make the output prettier on the eyes:</b>

```python
#!/usr/bin/python3
from json import dumps

d = {"string": "word", "dict": {"a": "a", "b": "b"}, "list": [0, 1, 2], }
#print the original:
print(dumps(d))
#print the pretty output:
print(dumps(d, indent=4, separators=(',', ': ')))
```

<b>JSON kwargs to sort the output:</b>

```python
#!/usr/bin/python3
from json import dumps

d = {"string": "word", "dict": {"a": "a", "b": "b"}, "list": [0, 1, 2], }
#print the original:
print(dumps(d))
#Make the output pretty and sort it:
print(dumps(d, indent=4, separators=(',', ': '), sort_keys= True))
```

<b>Dictify a web page</b>

```python
#!/usr/bin/python3
import json
from pprint import pprint
from urllib.request import urlopen

#open the web page:
html_content = urlopen('http://validate.jsontest.com/?json=%5BJSON-code-to-validate%5D').read()  

#load the JSON:
d_json_test = json.loads(html_content)

#print the JSON to screen:
pprint(d_json_test)

#in one line:
pprint(json.loads(urlopen('http://www.reddit.com/r/all/top/.json').read()))
```

<b>Using JSON in jinja</b>

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
#load json from string:
routers = json.loads(json_str)

#define template:
template = Template('''

   {% for router in routers %}
   set system host-name {{ router['hostname'] }}
   set interfaces lo0 unit 0 family inet address {{ router['mgmt-ip'] }} primary
   {% endfor %}

''')

# render template and send the output to screen:
print(template.render(routers = routers))
```

https://github.com/saidvandeklundert/saidvandeklundert.github.io/blob/workgin_with_json/_posts/2019-11-2-working-with-json.md