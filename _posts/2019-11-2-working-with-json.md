---
layout: post
title: Working with JSON in Python
tags: [automation, python]
image: /img/python-logo.png
---

JavaScript Object Notation, or JSON, is something that started popping up more and more the moment I started doing (Net)DevOps. I decided to combine the notes and examples I had gathered so far into this post.


JSON overview
=============

JSON is an open standard format that can be used to store or transmit data. 

The two data structures available in JSON are:
-	<b>Objects</b>: an unordered collection of one or more key-value pairs enclosed in braces {}
-	<b>Arrays</b>: and ordered collection of values enclosed in brackets, [].

An object is similar to a Python dictionary and an array is similar to a list in Python.

The values that you will find in these data structures are the following:
-	String: 0 or more Unicode characters
-	Number: an integer or float
-	true
-	false
-	null
-	Object
-	Array

JSON is case-sensitive, does not care about whitespaces and does not offer a way to put in any comments. Another thing that is nice to know is that every JSON file is also a valid YAML file.

The following is a table that show what happens when Python is encoded to JSON and when JSON is decoded to Python. The first column shows the Python data structures. The second column displays what values those data structures turn into when they are encoded into JSON. The last column shows the data structure that you'll end up with after you decode JSON to Python.


| Python data structures | encodes to JSON | decodes to Python |
| ---------------------- | --------------- | ----------------- |
| dict                   | object          | dict              |
| list, tuple            | array           | list              |
| str                    | string          | str               |
| int, float             | number          | int, float        |
| True                   | true            | True              |
| False                  | false           | False             |
| None                   | null            | None              |


Basic operations using JSON in Python
=====================================

There are multiple options available, but the only module I ever used is the one that is found in the Python Standard Library simply called [json](https://docs.python.org/3/library/json.html). The following examples all use this library.


<b>Store dictionary as json</b>


```python
#!/usr/bin/python3
from json import dump

d = {
    'string' : 'word',
    'integer' : 2,
    'float' : 2.15,
    'True' : True,
    'False' : False,
    'dict' : { 'a' : 'a', 'b' : 'b', },
    'list' : [ 0, 1, 2, ],
    'None' : None,
}
with open('dictionary.json', 'w') as f:
    dump(d, f)
```

<b>Load json from a file</b>

```python
#!/usr/bin/python3
from json import load

with open('dictionary.json', 'r') as f:    
    d = load(f)

print(d)
```

<b>Emit json as string</b>

```python
#!/usr/bin/python3
from json import load, dumps

with open('dictionary.json', 'r') as f:    
    d = load(f)

s = dumps(d)
print(s)

```

<b>dictify page</b>


```python
#!/usr/bin/python3
import json
from pprint import pprint
from urllib.request import urlopen

#open the web page:
html_content = urlopen('http://validate.jsontest.com/?json=%5BJSON-code-to-validate%5D').read()  

#load the json:
d_json_test = json.loads(html_content)

#print the json to screen:
pprint(d_json_test)

#in one line:
pprint(json.loads(urlopen('http://www.reddit.com/r/all/top/.json').read()))
```

<b>Load json in jinja</b>


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