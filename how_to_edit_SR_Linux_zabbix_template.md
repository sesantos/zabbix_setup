# How to craft Zabbix templates using JSON RPC

Zabbix templates allow to group useful items, triggers and other entities so that those can be reused again and again by applying to hosts in a single step.
When a template is linked to a host, the host inherits all entities of the template. So, basically a pre-prepared bunch of checks can be applied very quickly.
This guide is describing how to craft Zabbix templates for SR Linux and overall workflow you have to follow, plus additional tricks that help you tackle some cases.


## Step-by-step guide
1. _**Setup**_

First of all you need to create your own playground including Zabbix, some SR Linux setup, necessary toolkits and YANG models for your SRL version.
This samples are built using SR Linux version 21.6.2 and Zabbix 5.4, but it should be relevant for any new version at the extend YANG models for the new SRL version allowing to do so.

> Zabbix template referenced in this guide is [here](srlinux.lldp.yaml)

2. _**The key tools.**_

Number of tools can be used, but for this particular case we are using [containerlab](https://containerlab.srlinux.dev/) and [gnmic](https://gnmic.kmrd.dev). Of course, should have your Zabbix up and running and connectivity available between Zabbix containers/VM and mgmt interface of your SRL boxes.
```sh
[admin@dct0 6-nodes-pg]$ sudo clab  inspect -t 6npg.yml 
INFO[0000] Parsing & checking topology file: 6npg.yml   
+---+----------------+--------------+---------------------------------+-------+---------+----------------+----------------------+
| # |      Name      | Container ID |              Image              | Kind  |  State  |  IPv4 Address  |     IPv6 Address     |
+---+----------------+--------------+---------------------------------+-------+---------+----------------+----------------------+
| 1 | clab-6npg-cl1  | fcd8ddbdd036 | ghcr.io/hellt/network-multitool | linux | running | 172.20.20.8/24 | 2001:172:20:20::8/64 |
| 2 | clab-6npg-cl2  | 039c0a340bbb | ghcr.io/hellt/network-multitool | linux | running | 172.20.20.9/24 | 2001:172:20:20::9/64 |
| 3 | clab-6npg-srl1 | f5b055655952 | srlinux                         | srl   | running | 172.20.20.7/24 | 2001:172:20:20::7/64 |
| 4 | clab-6npg-srl2 | 7deac6f291e6 | srlinux                         | srl   | running | 172.20.20.3/24 | 2001:172:20:20::3/64 |
| 5 | clab-6npg-srl3 | 475ffd2d42d0 | srlinux                         | srl   | running | 172.20.20.2/24 | 2001:172:20:20::2/64 |
| 6 | clab-6npg-srl4 | 12f00dbbcc80 | srlinux                         | srl   | running | 172.20.20.6/24 | 2001:172:20:20::6/64 |
| 7 | clab-6npg-srl5 | 49ac2b8c5d2c | srlinux                         | srl   | running | 172.20.20.5/24 | 2001:172:20:20::5/64 |
| 8 | clab-6npg-srl6 | 03fcdef6b616 | srlinux                         | srl   | running | 172.20.20.4/24 | 2001:172:20:20::4/64 |
+---+----------------+--------------+---------------------------------+-------+---------+----------------+----------------------+
```
3. _**Connectivity verification.**_

From Zabbix you can verify everything is ok by running trivial check using ```curl```. So if you see HTTP error 405 it means you are goog to go with template development.

```curl
[admin@dczt0 ~]$ for n in `seq 2 7` ; do echo 172.20.20.$n; curl https://172.20.20.${n}/jsonrpc -k -i  | grep -i 'http/'; done
172.20.20.2
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  12714      0 --:--:-- --:--:-- --:--:-- 12714
HTTP/1.1 405 METHOD NOT ALLOWED
172.20.20.3
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  13692      0 --:--:-- --:--:-- --:--:-- 13692
HTTP/1.1 405 METHOD NOT ALLOWED
172.20.20.4
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  11125      0 --:--:-- --:--:-- --:--:-- 11125
HTTP/1.1 405 METHOD NOT ALLOWED
172.20.20.5
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  11866      0 --:--:-- --:--:-- --:--:-- 11866
HTTP/1.1 405 METHOD NOT ALLOWED
172.20.20.6
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  12714      0 --:--:-- --:--:-- --:--:-- 12714
HTTP/1.1 405 METHOD NOT ALLOWED
172.20.20.7
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   178  100   178    0     0  12714      0 --:--:-- --:--:-- --:--:-- 12714
HTTP/1.1 405 METHOD NOT ALLOWED
```

4. _**Template creation.**_

Overall template creation is decribed very well by official Zabbix documentation, so we wil focus on the workflow and nuances related to that. 
For the newly created template we have to setup a suitable name and assign it to group (should be pre-created, if you would like to use a new one).
To be able to link teamplate with to parent one we have to some unique name, that's why ```nokia.netops.srl.lldp``` is used to reflect yang model covered by particular template.
![New teamplate][new_template]

5. _**Macros.**_

The next step should be quite common for any template supposed to be created for SR Linux. The idea is simple - you are creating all necessary variables and default values in order to parametrize connectivity params like protocol (HTTP or HTTPS), port, user and password (you could override them in host level, if necessary).

![Template macro][template_macro]

User macro are well described in the [official documentation](https://www.zabbix.com/documentation/current/manual/config/macros/user_macros).

Marcos actively used for URL specification of HTTP items, since it should purely dynamic:

```
{$PROTO}://{HOST.CONN}:{$JSONPORT}/jsonrpc
```

6. _**YANG models.**_

For the sake of example we will use LLDP data model. So, [pyang](https://github.com/mbj4668/pyang) can be used to generate nice view and improve your navigation experience.
![Yang tree][yang_tree]

If you have your yang models under some folder you have to generate list of directories to search modules by pyang tool. 
```sh
[admin@dct0 6-nodes-pg]$ more /home/admin/bin/lod.sh
#!/bin/bash
ls -R -m | grep \: | sed ':x; /./ {N; s/\n//g; bx}' | sed -e "s/\\:/${1}/g;"
```
Then go to the directory tree root, specify module names required and generate HTML tree.
In this case all modules were chosen to generate full tree. Of course, for ```log.sh``` colon is used as parameter as a separator for the list of directories where pyang will try to find modules.
```sh
[admin@dct0 models]$ SRL_MOD=`find ./ -type f | grep yang | sed ':x; /./ {N; s/\n/ /g; bx}'`
[admin@dct0 models]$ pyang -p `lod.sh \:` -f jstree -o 21.6.2.tree.html $SRL_MOD
```

7. _**Items.**_

If you need to monitor specific parameter in the model, then you can create an item directly under template w/o specifying discovery rules. As soon as we are using JSON RPC, HTTP item type gonna be extensively used together with JSONpath preprocessing
![Template items][items_under_template]

In this particular case admin state of LLDP protocol supposed to be monitored.
More detailed documentation about JSON RPC methods can be found in [System Management Guide](https://infocenter.nokia.com/public/SRLINUX216R1A/index.jsp?topic=%2Fcom.srlinux.sysmgmt%2Fhtml%2Fjson-interface.html&cp=10_4_4_0&anchor=lindapra5iwesiyjns0). 
HTTP method is POST and the following JSON is used to retrieve LLDP info from state datastore.
```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "get",
  "params": {
      "commands": [
          {
              "path": "/system/lldp",
              "datastore": "state",
              "recursive": true
          }
      ]
  }
}
```

8. _**Preprocessing.**_

Item preprocessing need to exact required variables and values from received JSON.

> Important! 
> In case you are doing JSONpath filtering in preprocessing yu have to add additional tag ```body``` right after ```$```, like ```$.body.result[0].["admin-state"]```, but not like under discovery rules.


![Item preprocessing][item_preprocessing_1]

In this particular case we are extracting admin-state value and translating it to numeric representation to have nice graph at the end.

More info about item preprocessing and available functions can be found [here](https://www.zabbix.com/documentation/current/manual/config/items/preprocessing), but we will touch most essential of them in this document. 


9. _**Additional item example**_

To demonstrate case more extensively we can consider the following case.
Let's imagine you want to make sure number of active LLDP interface remains the of has been changed due to some circumstances or just to get informational triggers about the change.

![# of LLDP inf][lldp_number_inf]

JSON RPC request is the following:
```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "get",
  "params": {
      "commands": [
          {
              "path": "/system/lldp",
              "datastore": "state",
              "recursive": true
          }
      ]
  }
}
```
As result you will get something like can be found below, which require additional processing before something suitable for use can unutilized.

```sh
(srlJSONAPI-pg) azyablov@mbp srlJSONAPI-pg % curl https://admin:admin@localhost:11001/jsonrpc --insecure -H "Content-Type:application/json" -d @curl/get_system.lldp.json  | python3.7 -m json.tool 
```
```json
{
    "result": [
        {
            "admin-state": "enable",
            "hello-timer": "30",
            "hold-multiplier": 4,
            "system-name": "srl1",
            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl1 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
            "bgp-auto-discovery": {
                "admin-state": "enable"
            },
            "statistics": {
                "frame-in": "536088",
                "frame-out": "320838",
                "frame-error-in": "0",
                "frame-discard": "0",
                "tlv-discard": "0",
                "tlv-unknown": "0",
                "tlv-accepted": "3887670",
                "entries-aged-out": "0"
            },
            "interface": [
                {
                    "name": "ethernet-1/1",
                    "admin-state": "enable",
<...snip...>
                        {
                            "id": "02:D0:98:FF:00:00",
                            "first-message": "2021-10-27T13:16:53.466Z",
                            "last-update": "2021-11-24T12:26:45.576Z",
                            "system-name": "srl2",
                            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl2 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
                            "chassis-id": "02:D0:98:FF:00:00",
                            "chassis-id-type": "MAC_ADDRESS",
                            "port-id": "mgmt0",
                            "port-id-type": "INTERFACE_NAME",
                            "port-description": "",
                            "capability": [
                                {
                                    "name": "Router",
                                    "enabled": true
                                }
                            ]
                        }
                    ]
                }
            ]
        }
    ],
    "id": 0,
    "jsonrpc": "2.0"
}

```

Very simple and elegant JSONpath function is coming to help.
More about [JSONpath functionality](https://www.zabbix.com/documentation/current/manual/config/items/preprocessing/jsonpath_functionality).
```
$.body.result[0].interface.length()
```
![JSONpath][jsonpath_func]

10. _**Triggers**_

Triggers allow to identify almost any kind of problem and deviation.
Zabbix [official documentation](https://www.zabbix.com/documentation/current/manual/config/triggers/expression) provide good explanation and as well as reach set of various [functions](https://www.zabbix.com/documentation/current/manual/appendix/functions) to manipulate/calculate/transform/test item values and even historical trends changes.

For example, alarm warning severity will be triggered in case LLDP administratively down more than 30 min (in case it deactivated by mistake or incorrectly set by automation).

Problem expression: ```last(/nokia.netops.srl.lldp/nokia.netops.srl.lldp.admin,#31)=0```
Recovery expression: ```last(/nokia.netops.srl.lldp/nokia.netops.srl.lldp.admin,#1)=1```

![Trigger][trigger]

11. _**Discovery rules**_

Coming to more sophisticated topic how to discover and monitor hierarchical structures and lists of dynamic objects/entities. In order to solve it Zabbix is providing [low level discovery rules](https://www.zabbix.com/documentation/current/manual/discovery/low_level_discovery).

![Discovery rules][discovery_rules]


The idea of discovery rules are indented to discover entities provided by NE as a list or array. Then automatically create items, triggers, and graphs for these entities. In this case you do not need to add manually them and you can start monitoring them automatically.
More detailed information can be found in [official documentation](https://www.zabbix.com/documentation/current/manual/discovery/low_level_discovery)

As soon items ara dynamic entities you need some way to create unique keys, substitute fields with real values in order to create item, triggers and graphs.
LLD marcos are used to perform that tasks and very similar to yser marco, but start with ```#``` instead of ```$```.

In our example we are going to monitor some interface related values, like frame-in/out rate.

![Discovery rules LLDP inf][discovery_rule_lldp_inf]

JSON is quite straightforward:

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "get",
  "params": {
      "commands": [
          {
              "path": "/system/lldp/interface[name=*]",
              "datastore": "state",
              "recursive": true
          }
      ]
  }
}
```
In preprocessing section in one step to extract list of interfaces using ```JSONpath```:

```go
$.result[0].interface
```
In the next step we have to extract necessary LLD marco to be able successfully build items, triggers, etc.

![LLD marco][lld_marco]

> ! Pay attention, that here we are not referring to some nested list and/or dynamic entities, it will be discussed later, since require special approach.

In our case we are going to use filter, because not all interfaces we are going to enable LLDP protocol, as such we need to get rid out of them by using filter and leave only mgmt, ethernet interfaces, which are administratively enabled and UP.
As per [official Zabbix documentation](https://www.zabbix.com/documentation/current/manual/discovery/low_level_discovery):
>"A filter can be used to generate real items, triggers, and graphs only for entities that match the criteria."

![Discovery rule filters][discovery_rule_filters]

12. _**Item prototypes**_

As soon as discovery rules are defined and necessary macro developed, we can proceed with creation of items, triggers, etc... Let's take relatively simple case, where ```frame-rate-in/out``` rate of LLDP interface is monitored in order to detect abnormal behavior.
JSON RPC request is defined with marco ```{#NAME}``` to catch up necessary interface, as well as the item key ```nokia.netops.srl.lldp.interfaces.statistics.frame-rate-in.[{#NAME}]```

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "get",
  "params": {
      "commands": [
          {
              "path": "/system/lldp/interface[name={#NAME}]/statistics/frame-in",
              "datastore": "state",
              "recursive": true
          }
      ]
  }
}
```
![LLDP frame in rate][lldp_frame_in]

Preprocessing section has two steps:
* Extact interface data ```$.result[0]```
* Calculate simple change between to consecutive values [in this case update interval is 30s]

![item prototype preprocessing][item_proto_preprocessing]

## Advanced preprocessing of hieranhical and dynamic JSON entities 

This section describes advanced preprocessing of retrieved JSON data with hierarchical and dynamic entities. For the sake of example LLDP interface neighbors will be considered. We will discover all LLDP neighbors per interfaces and track uptime.
In containerlab environment we have single bridge where all mgmt interfaces of SRL containers attached, which makes our exercise even more interesting and representative. By retrieving JSON by using the following RPC request built in accordance with LLDP YANG model
```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "get",
  "params": {
      "commands": [
          {
              "path": "/system/lldp/interface[name=*]/neighbor[id=*]",
              "datastore": "state",
              "recursive": true
          }
      ]
  }
}
```
, system should receive something similar to the following:
```json
{
                    "name": "mgmt0",
                    "admin-state": "enable",
                    "oper-state": "up",
                    "statistics": {
                        "frame-in": "401034",
                        "frame-out": "80212",
                        "frame-error-in": "0",
                        "frame-discard": "0",
                        "tlv-discard": "0",
                        "tlv-unknown": "0",
                        "frame-error-out": "0"
                    },
                    "neighbor": [
                        {
                            "id": "02:11:CA:FF:00:00",
                            "first-message": "2021-10-27T13:16:53.466Z",
                            "last-update": "2021-11-24T12:26:45.574Z",
                            "system-name": "srl4",
                            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl4 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
                            "chassis-id": "02:11:CA:FF:00:00",
                            "chassis-id-type": "MAC_ADDRESS",
                            "port-id": "mgmt0",
                            "port-id-type": "INTERFACE_NAME",
                            "port-description": "",
                            "capability": [
                                {
                                    "name": "Router",
                                    "enabled": true
                                }
                            ]
                        },
                        {
                            "id": "02:78:3B:FF:00:00",
                            "first-message": "2021-10-27T13:16:53.466Z",
                            "last-update": "2021-11-24T12:26:45.576Z",
                            "system-name": "srl5",
                            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl5 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
                            "chassis-id": "02:78:3B:FF:00:00",
                            "chassis-id-type": "MAC_ADDRESS",
                            "port-id": "mgmt0",
                            "port-id-type": "INTERFACE_NAME",
                            "port-description": "",
                            "capability": [
                                {
                                    "name": "Router",
                                    "enabled": true
                                }
                            ]
                        },
                        {
                            "id": "02:A4:91:FF:00:00",
                            "first-message": "2021-10-27T13:16:53.466Z",
                            "last-update": "2021-11-24T12:26:45.574Z",
                            "system-name": "srl6",
                            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl6 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
                            "chassis-id": "02:A4:91:FF:00:00",
                            "chassis-id-type": "MAC_ADDRESS",
                            "port-id": "mgmt0",
                            "port-id-type": "INTERFACE_NAME",
                            "port-description": "",
                            "capability": [
                                {
                                    "name": "Router",
                                    "enabled": true
                                }
                            ]
                        },
                        {
                            "id": "02:C5:44:FF:00:00",
                            "first-message": "2021-10-27T13:16:53.466Z",
                            "last-update": "2021-11-24T12:26:45.574Z",
                            "system-name": "srl3",
                            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl3 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
                            "chassis-id": "02:C5:44:FF:00:00",
                            "chassis-id-type": "MAC_ADDRESS",
                            "port-id": "mgmt0",
                            "port-id-type": "INTERFACE_NAME",
                            "port-description": "",
                            "capability": [
                                {
                                    "name": "Router",
                                    "enabled": true
                                }
                            ]
                        },
                        {
                            "id": "02:D0:98:FF:00:00",
                            "first-message": "2021-10-27T13:16:53.466Z",
                            "last-update": "2021-11-24T12:26:45.576Z",
                            "system-name": "srl2",
                            "system-description": "SRLinux-v21.6.2-67-g30bcd78867 srl2 3.10.0-1160.24.1.el7.x86_64 #1 SMP Thu Apr 8 19:51:47 UTC 2021",
                            "chassis-id": "02:D0:98:FF:00:00",
                            "chassis-id-type": "MAC_ADDRESS",
                            "port-id": "mgmt0",
                            "port-id-type": "INTERFACE_NAME",
                            "port-description": "",
                            "capability": [
                                {
                                    "name": "Router",
                                    "enabled": true
                                }
                            ]
                        }
                    ]
                }

```
![LLDP neighors][lldp_neighbors]

The key and name built using LLD macro:
```nokia.netops.srl.lldp.neighbors.[{#INFNAME},{#NEIGHBORSYSTEMNAME}]```

![LLDP neighbors LLD macro][lldp_nei_lld_marco]

Filters are configured as per interest to check only mgmt and ethernet interfaces:

![LLDP neighbors filter][lldp_nei_filter]


As easy to notice JSONpath of LLD macros are not corresponding to YANG model or JSON structure, because additional preprocessing were applied to flatten received JSON data, otherwise it's not possible to create items for nested dynamic entities, just because Zabbix expect list or array of items and additional preprocessing supposed to be done before JSON data can be used for LLD macro value assignment. This task can be performed via JavaScript preprocessing. 

![LLDP neighbors preprocessing JS][lldp_nei_preprocessing_js]

In this case code of JavaScript function supposed to be provided as content of preprocessing step, while retrieved JSON is provided in ```value``` variable to that function.

![LLDP nei. JS][lldp_nei_js]

```javascript
    // create an empty array to flatten info about LLDP neighbors
    var lldpNeighbors = []; 

    var receivedJSON= JSON.parse(value);
    for (i = 0; i < receivedJSON.length; i++) {
        if (receivedJSON[i].hasOwnProperty("neighbor")){
            var name = receivedJSON[i].name;
            var neighbors = receivedJSON[i].neighbor;
            // for debug
            //console.log(i)
            for (n = 0; n < neighbors.length; n++) {
                lldpNeighbors.push({
                    infname: name, 
                    neighborid: neighbors[n].id,
                    neighborsystemname: neighbors[n]["system-name"],
                    firstmessage: neighbors[n]["first-message"],
                    lastupdate: neighbors[n]["last-update"],
                })
                
            }

        }

    }
    // for debug
    // console.log(JSON.stringify(lldpNeighbors))
    return JSON.stringify(lldpNeighbors);
```

As result script return simple JSON that can be used to retrieve values for LLD macro.
In the next turn item prototypes can be created. In our case we retrieve LLDP neighbor state and calculate uptime.
The item key is and JSON RPC are using LLD macro from previous step:

```nokia.netops.srl.lldp.neighbors.uptime.[{#INFNAME},{#NEIGHBORSYSTEMNAME}]``` 

```json
{
  "jsonrpc": "2.0",
  "id": 0,
  "method": "get",
  "params": {
      "commands": [
          {
              "path": "/system/lldp/interface[name={#INFNAME}]/neighbor[id={#NEIGHBORID}]",
              "datastore": "state",
              "recursive": true
          }
      ]
  }
}
```

But here we are not receiving just an uptime value, but time when first LLDP message was received and the last one from neighbor:

```json
    "first-message": "2021-10-27T13:16:53.466Z",
    "last-update": "2021-11-24T12:26:45.574Z",
```
But additional preprocessing is possible to do necessary calculations by using JavaScript and return human friendly value:
```javascript
var receivedJSON= JSON.parse(value);

    const fm = new Date(receivedJSON["first-message"]);
    const lu = new Date(receivedJSON["last-update"]);
    const diffTime = Math.abs(lu - fm);
    const uptimed = Math.trunc(diffTime / 1000 / 3600 / 24);
    const uptimeh = Math.trunc(diffTime / 1000 / 3600 -  24 * uptimed);
    const uptimem = Math.trunc(diffTime / 1000 / 60  -  60 * uptimeh -  24 * 60 * uptimed);
    const uptimes = Math.trunc(diffTime / 1000 -  3600 * uptimeh -  24 * 3600 * uptimed - 60 * uptimem);;
    const uptime = uptimed.toString() + " Days " + uptimeh.toString() + " Hours " + uptimem.toString() + " Min " + uptimes.toString() + " Sec"
    // debug
    // console.log(uptime)
                
    return JSON.stringify(uptime);
```
Finally, in the latest data various template discovery items will appear after a few minutes:

![latest data][latest_data]

[new_template]: pic/new_template.png "New template"
[template_macro]: pic/template_macro.png "Template macro"
[yang_tree]: pic/yang_tree.png "Yang tree"
[items_under_template]: pic/items_under_template.png "Template items"
[item_preprocessing_1]: pic/item_preprocessing_1.png "Item preprocessing"
[lldp_number_inf]: pic/lldp_number_inf.png "# of LLDP inf"
[jsonpath_func]: pic/jsonpath_func.png "JSONpath"
[discovery_rules]: pic/discovery_rules.png "Discovery rules"
[discovery_rule_lldp_inf]: pic/discovery_rule_lldp_inf.png "Discovery rule LLDP inf"
[lld_marco]: pic/lld_marco.png "LLDP marco"
[discovery_rule_filters]: pic/discovery_rule_filters.png "LLDP marco"
[lldp_frame_in]: pic/lldp_frame_in.png "LLDP frame in rate"
[item_proto_preprocessing]: pic/item_proto_preprocessing.png "Item prototype preprocessing"
[trigger]: pic/trigger.png "Trigger"
[lldp_neighbors]: pic/lldp_neighbors.png "LLDP neighbors"
[lldp_nei_lld_marco]: pic/lldp_nei_lld_marco.png "LLDP neighbors LLD macro"
[lldp_nei_filter]: pic/lldp_nei_filter.png "LLDP neighbors filter"
[lldp_nei_preprocessing_js]: pic/lldp_nei_preprocessing_js.png "LLDP neighbors preprocessing JS"
[lldp_nei_js]: pic/lldp_nei_js.png "LLDP neighbors JS"
[latest_data]: pic/latest_data.png "Latest data"