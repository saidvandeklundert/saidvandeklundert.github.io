---
layout: post
title: JSON with Juniper PyEZ
image: /img/juniper_logo.jpg
---


By default, <b>Juniper</b> will return <b>XML</b> in response to your RPC calls. But if you prefer, Juniper can translate the XML into <b>JSON</b> for you. 

When using <b>PyEZ</b>, all you have to do is pass a dictionary (`{'format':'json'}`) to the RPC call. Here is how an example script could look like:

```python
from pprint import pprint

def get_ospf_neighbor_information(username, pwd, host ):
    dev = Device(host=host, user=username, password=pwd)
    dev.open()
    rpc = dev.rpc.get_ospf_neighbor_information({'format':'json'}, )
    dev.close()
    return rpc


if __name__ == "__main__": 
    

    ospf_r1 = get_ospf_neighbor_information('salt-r1', 'salt123', '169.50.169.171' )
    pprint(ospf_r1)
```


Let’s run the script and examine the information that is returned. To this end, we pass `-i` to enter interactive mode after executing the script like so:

```python
[said@srv ]$ python -i json-ospf.py
{u'ospf-neighbor-information': [{u'ospf-neighbor': [{u'activity-timer': [{u'data': u'35'}],
                                                     u'interface-name': [{u'data': u'ge-0/0/1.1'}],
                                                     u'neighbor-address': [{u'data': u'192.168.1.1'}],
                                                     u'neighbor-id': [{u'data': u'10.0.0.2'}],
                                                     u'neighbor-priority': [{u'data': u'128'}],
                                                     u'ospf-neighbor-state': [{u'data': u'Full'}]},
                                                    {u'activity-timer': [{u'data': u'35'}],
                                                     u'interface-name': [{u'data': u'ge-0/0/1.4'}],
                                                     u'neighbor-address': [{u'data': u'192.168.4.1'}],
                                                     u'neighbor-id': [{u'data': u'10.0.0.4'}],
                                                     u'neighbor-priority': [{u'data': u'128'}],
                                                     u'ospf-neighbor-state': [{u'data': u'Full'}]}]}]}
>>>
```

After seeing the output from the `pprint` statement, we can start examining what the device returned:

```python
>>> type(ospf_r1)
<type 'dict'>
>>> 
```


PyEZ turns the `JSON` as a `python dictionary` for us. In interactive mode, we can gradually drill our way down to the things we are really interested in:

```python
>>> pprint(ospf_r1['ospf-neighbor-information'])
[{u'ospf-neighbor': [{u'activity-timer': [{u'data': u'35'}],
                      u'interface-name': [{u'data': u'ge-0/0/1.1'}],
                      u'neighbor-address': [{u'data': u'192.168.1.1'}],
                      u'neighbor-id': [{u'data': u'10.0.0.2'}],
                      u'neighbor-priority': [{u'data': u'128'}],
                      u'ospf-neighbor-state': [{u'data': u'Full'}]},
                     {u'activity-timer': [{u'data': u'33'}],
                      u'interface-name': [{u'data': u'ge-0/0/1.4'}],
                      u'neighbor-address': [{u'data': u'192.168.4.1'}],
                      u'neighbor-id': [{u'data': u'10.0.0.4'}],
                      u'neighbor-priority': [{u'data': u'128'}],
                      u'ospf-neighbor-state': [{u'data': u'Full'}]}]}]
>>> type(ospf_r1['ospf-neighbor-information'])      
<type 'list'>
>>> 
```

When we access the `ospf-neighbor-information` key, we get a list. From the `pprint` output, we can also see that the list containts 1 item, a dictionary with a key called `ospf-neighbor`:

```python
>>> type(ospf_r1['ospf-neighbor-information'][0])                 
<type 'dict'>
>>> 
>>> pprint(ospf_r1['ospf-neighbor-information'][0]['ospf-neighbor'])  
[{u'activity-timer': [{u'data': u'35'}],
  u'interface-name': [{u'data': u'ge-0/0/1.1'}],
  u'neighbor-address': [{u'data': u'192.168.1.1'}],
  u'neighbor-id': [{u'data': u'10.0.0.2'}],
  u'neighbor-priority': [{u'data': u'128'}],
  u'ospf-neighbor-state': [{u'data': u'Full'}]},
 {u'activity-timer': [{u'data': u'33'}],
  u'interface-name': [{u'data': u'ge-0/0/1.4'}],
  u'neighbor-address': [{u'data': u'192.168.4.1'}],
  u'neighbor-id': [{u'data': u'10.0.0.4'}],
  u'neighbor-priority': [{u'data': u'128'}],
  u'ospf-neighbor-state': [{u'data': u'Full'}]}]
>>> 
```

This contains a list of dictionaries that describe the individual neighbors. We can step through the list and check the information like this:

```python
>>> for neighbor in ospf_r1['ospf-neighbor-information'][0]['ospf-neighbor']:
...     print('OSPF neighbor: {}'.format(neighbor['neighbor-id'][0]['data']))
...     print(neighbor['neighbor-address'][0]['data'])                                   
...     print(neighbor['interface-name'][0]['data'])                                     
...     print(neighbor['ospf-neighbor-state'][0]['data'])  
... 
OSPF neighbor: 10.0.0.2
192.168.1.1
ge-0/0/1.1
Full
OSPF neighbor: 10.0.0.4
192.168.4.1
ge-0/0/1.4
Full
>>> 
```

In case we want to store this information for future use, we just dump it as `json`:


```python
>>> from json import dumps
>>> pprint(my_json) 
'{"ospf-neighbor-information": [{"ospf-neighbor": [{"neighbor-address": [{"data": "192.168.1.1"}], "neighbor-priority": [{"data": "128"}], "interface-name": [{"data": "ge-0/0/1.1"}], "neighbor-id": [{"data": "10.0.0.2"}], "activity-timer": [{"data": "35"}], "ospf-neighbor-state": [{"data": "Full"}]}, {"neighbor-address": [{"data": "192.168.4.1"}], "neighbor-priority": [{"data": "128"}], "interface-name": [{"data": "ge-0/0/1.4"}], "neighbor-id": [{"data": "10.0.0.4"}], "activity-timer": [{"data": "33"}], "ospf-neighbor-state": [{"data": "Full"}]}]}]}'
>>> with open('ospf.json', 'w') as file:
...    file.write(dumps(ospf_r1, indent=4, separators=(',', ': ')))
... 
>>> 
```

When we want to access the data later on, we can use the following to load the file as a dictionary again:

```python
>>> from json import load
>>> with open("ospf.json", "r") as file:
...     ospf_json = load(file)
... 
>>> ospf_json
{u'ospf-neighbor-information': [{u'ospf-neighbor': [{u'neighbor-address': [{u'data': u'192.168.1.1'}], u'neighbor-priority': [{u'data': u'128'}], u'interface-name': [{u'data': u'ge-0/0/1.1'}], u'neighbor-id': [{u'data': u'10.0.0.2'}], u'activity-timer': [{u'data': u'39'}], u'ospf-neighbor-state': [{u'data': u'Full'}]}, {u'neighbor-address': [{u'data': u'192.168.4.1'}], u'neighbor-priority': [{u'data': u'128'}], u'interface-name': [{u'data': u'ge-0/0/1.4'}], u'neighbor-id': [{u'data': u'10.0.0.4'}], u'activity-timer': [{u'data': u'38'}], u'ospf-neighbor-state': [{u'data': u'Full'}]}]}]}
>>> 
```


When we leave the interpreter, we can check out the file we just created like so:

```json
[said@srv ]$ cat ospf.json 
{
    "ospf-neighbor-information": [
        {
            "ospf-neighbor": [
                {
                    "neighbor-address": [
                        {
                            "data": "192.168.1.1"
                        }
                    ],
                    "neighbor-priority": [
                        {
                            "data": "128"
                        }
                    ],
                    "interface-name": [
                        {
                            "data": "ge-0/0/1.1"
                        }
                    ],
                    "neighbor-id": [
                        {
                            "data": "10.0.0.2"
                        }
                    ],
                    "activity-timer": [
                        {
                            "data": "35"
                        }
                    ],
                    "ospf-neighbor-state": [
                        {
                            "data": "Full"
                        }
                    ]
                },
                {
                    "neighbor-address": [
                        {
                            "data": "192.168.4.1"
                        }
                    ],
                    "neighbor-priority": [
                        {
                            "data": "128"
                        }
                    ],
                    "interface-name": [
                        {
                            "data": "ge-0/0/1.4"
                        }
                    ],
                    "neighbor-id": [
                        {
                            "data": "10.0.0.4"
                        }
                    ],
                    "activity-timer": [
                        {
                            "data": "33"
                        }
                    ],
                    "ospf-neighbor-state": [
                        {
                            "data": "Full"
                        }
                    ]
                }
            ]
        }
    ]
}
```


Not a fan of seeing lists in dicts in lists in dicts? Consider the following:

```bash
set system export-format state-data json compact
```


This is available starting 17.3R1 and will have the device emit compact `JSON` format.  After configuring the compact format, this is how the output from R1 would look like:

```json
salt@vmx01:r1> show ospf neighbor |display json    
{
    "ospf-neighbor-information" :
    {
        "ospf-neighbor" :
        {
            "neighbor-address" : "192.168.1.1", 
            "interface-name" : "ge-0/0/1.1", 
            "ospf-neighbor-state" : "Full", 
            "neighbor-id" : "10.0.0.2", 
            "neighbor-priority" : "128", 
            "activity-timer" : "38"
        }, 
        {
            "neighbor-address" : "192.168.4.1", 
            "interface-name" : "ge-0/0/1.4", 
            "ospf-neighbor-state" : "Full", 
            "neighbor-id" : "10.0.0.4", 
            "neighbor-priority" : "128", 
            "activity-timer" : "35"
        }
    }
}
```


Hope this helps you get started with PyEZ and JSON.