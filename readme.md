
# Introduction

 This page  will guide you through the installation and configuration of a Zabbix instance for Nokia SR Linux device managament.<br>
 A Zabbix template for SR Linux devices is available [here](https://github.com/sesantos/zabbix_sdk/blob/master/templates/srlinux_template.yaml) <br>
 If you need to edit or change the template an example is presented [here](https://github.com/azyablov/crafting_srl_zabbix_template#readme) <br><br>


# Download docker-compose 
```
curl -L https://github.com/docker/compose/releases/download/v2.1.0/docker-compose-linux-x86_64 -o /usr/local/bin/docker-compose
ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose 
sudo chmod +x /usr/local/bin/docker-compose
```

# Clone Zabbix-Docker repo

```
git clone https://github.com/zabbix/zabbix-docker.git
cd zabbix-docker
git checkout -b 5.4
mkdir -p zbx_env/usr/share/zabbix/modules
```

# Create a Zabbix instance

```
docker-compose -f docker-compose_v3_ubuntu_mysql_latest.yaml up -d
```

![Alt text](/gifs/zabbix-docker-compose.gif)

# Disable TCP Checksum offload on all interfaces connected to zabbix docker  bridges
```
for i in `sudo docker network ls -f NAME=zabbix* -q `; do for j in `sudo bridge link | grep $i |  cut -d : -f 2 | cut -d @ -f 1`;do sudo ethtool --offload $j rx off tx off; done; done
```

# Connect to your Zabbix Instance on HTTP web interface

### Login with:
username: ***Admin*** <br>
password: ***zabbix***

![Alt text](/gifs/zabbix-login.gif)

# Setup the Zabbix Instance for SR Linux management 
### Clone SR Linux Zabbix Template and Setup Script

```
git clone https://github.com/sesantos/zabbix_sdk.git
python3.6 setup.py install
```

The Zabbix SR Linux template is available [here](https://github.com/sesantos/zabbix_sdk/blob/master/templates/srlinux_template.yaml) under /templates

### Load the SR Linux template and setup the Zabbix for node discovery

It can be loaded using the Web Gui under Templates or using the zabbix_sdk.

When using the zabbix_sdk the template, discovery rule and discovery action are loaded. 
The ***setup.yaml*** provides the definition of the SR Linux setup and zabbix environment.

```
zabbix_instance:
    ip: localhost
    port: 80 #default is 8080
    user: Admin
    password: zabbix
srlinux_setup:
    ip_range: "172.18.18.4-6"
    user_name: "admin"
    password: "admin"
    json_rpc_port: "80"
    proto: "http"
    snmp_community: "public"

```

When the parameters of ***setup.yaml*** are all set the template is loaded and the Zabbix instance is configured to discover the SR Linux devices:

 ```
 zabbix_setup_srlinux_env -f templates/ -s setup.yaml
 ```

 ![Alt text](/gifs/zabbix-load-template.gif)


## Load the SR Linux Template over Zabbix Web Interface and Configure Discovery Rules and Discovery Actions

1-Load the template under: ***Configuration -> Templates -> Import*** <br>
2-Update the template password for SR Linux admin user login

![Alt text](/gifs/zabbix-load-template-GUI.gif) <br>


3 - Create a discovery rule for SR Linux under: ***Configuration -> Discovery -> Create Discovery rule***

![Alt text](/gifs/zabbix-create-drule.gif) <br>


4 - Create a discovery action under: ***Configuration -> Actions -> Discovery Actions -> Create Discovery Action***

![Alt text](/gifs/zabbix-create-drule-action.gif) <br>

# Verify that SR Linux devices are discovered

1 - Verify devices and managed objecrts under: ***Configuration -> Hosts*** <br>
2 - Verify the device data under: ***Monitoring -> Hosts***

![Alt text](/gifs/zabbix-srlinux-device.gif) <br>

