# IXO node monitoring tool

To monitor you node your should have installed and configured:
On node server:
* [IXO node](https://github.com/ixofoundation/genesis/blob/master/impacthub-3/README_INSTALL.md) which should be configured (correct moniker, validator key, network ports setup)
* [Telegraf agent](https://www.influxdata.com/time-series-platform/telegraf/)
* [mon_ixo](https://github.com/svv28/mon_IXO) scripts set

On monitoring server:
* [InfluxDB](https://www.influxdata.com/products/influxdb/)
* [Grafana](https://grafana.com/)

It is possible to install the software on the node server instance. Hovewer, it is better to move it to standalone instance with opened web access to watch it from browser at any location.

## To use our free monitoring servise on pro-nodes.com:

just go to the section [**Installation on a node**](https://github.com/svv28/mon_IXO#installation-on-a-node) and follow the installation process.

Advantages  of using our free service:
* Our monitoring service is working on dedicated server (24/7 online)
* No need to install database  (InfluxDB)
* No need to install and configure  Grafana Dashboard
* On Grafana dashboard you will find all necessary metrics of your node (we use this monitoring service by ourselves, so we've configured dashboard properly)


## The following steps will guide you through the setup process (to configure on your own server):

### Monitoring server installation 

#### InfluxDB 

Install:
```
wget -qO- https://repos.influxdata.com/influxdb.key | gpg --dearmor | sudo tee /etc/apt/trusted.gpg.d/influxdb.gpg > /dev/null
export DISTRIB_ID=$(lsb_release -si); export DISTRIB_CODENAME=$(lsb_release -sc)
echo "deb [signed-by=/etc/apt/trusted.gpg.d/influxdb.gpg] https://repos.influxdata.com/${DISTRIB_ID,,} ${DISTRIB_CODENAME} stable" | sudo tee /etc/apt/sources.list.d/influxdb.list > /dev/null

sudo apt update && sudo apt install influxdb

sudo systemctl enable --now influxdb

sudo systemctl start influxdb

sudo systemctl status influxdb
```

Setup database (change the passwords given in the example on more secure ones):
```
influx
> create database ixometricsdb
> create user metrics with password 'password'
> grant WRITE on ixometricsdb to metrics
> create user grafana with password 'other_password'
> grant READ on ixometricsdb to grafana
```

Keep database user and password in order to use it later for agent configuration. Write it. 

In the case of using standalone instance for monitoring staff,  you should know your node external ip address (you can know it by command ```curl ifconfig.me```).
In the case of installation on the same instance, just use **localhost** or **127.0.0.1**

#### Grafana
Install:
```
wget -q -O - https://packages.grafana.com/gpg.key | sudo apt-key add -
sudo add-apt-repository "deb https://packages.grafana.com/oss/deb stable main"

sudo apt update -y
sudo apt install grafana -y

sudo systemctl daemon-reload

sudo systemctl enable --now grafana-server
sudo systemctl start grafana-server

# verify the status of the Grafana service with the following command:
sudo systemctl status grafana-server
```
Configuration:

Follow  **YOUR_MONITORING_SERVER_IP:3000** to setup grafana dashboard.
The following steps are performed in the graphical interface of grafana.

Change default password for grafana user admin/admin on safer one

Add data source InfluxDB with the following settings:


| Param            | Value                 | 
| ---------------- |:---------------------:|
| HTTP             |                       |
| URL              | http://localhost:8086 |
| InfluxDB Details |                       |
| Database         | ixometricsdb         |
| User             | grafana               |

Save datasource settings 

Import [json file](https://raw.githubusercontent.com/svv28/mon_IXO/main/mon_ixo-grafana-dashboard.json) from this repo and save your dashboard.

### Installation on a node

#### By fast installation script

You can use fast installation script.

**IMPORTANT**: You sholud run the script under the user where it is installed IXO node.

Don't use **sudo** if IXO-user is not a **root** 
```
wget https://raw.githubusercontent.com/svv28/mon_IXO/main/install.sh
cd mon_IXO
chmod +x install.sh
./install.sh
```
It will install telegraf agent, clone project repo and extract your node data as MONIKER, VALOPER ADDR, RPC PORT.
You should answer some questions about your monitoring service from part **Monitoring server installation**

**IF you would use our free monitoring service for you IXO node, you should enter next parameters:**
```
 urls = [ "http://pro-nodes.com:8086" ] # example http://yourownmonitoringnode:8086
 username = "metrics" # your database username
 password = "password" # your database user's password
```



#### Manual installation of telegraf and monitoring script

Install telegraf
```
sudo apt update
sudo apt -y install curl jq bc

# install telegraf
sudo cat <<EOF | sudo tee /etc/apt/sources.list.d/influxdata.list
deb https://repos.influxdata.com/ubuntu bionic stable
EOF
sudo curl -sL https://repos.influxdata.com/influxdb.key | sudo apt-key add -

sudo apt update
sudo apt -y install telegraf

sudo systemctl enable --now telegraf
sudo systemctl is-enabled telegraf

# make the telegraf user sudo and adm to be able to execute scripts as IXO user
sudo adduser telegraf sudo
sudo adduser telegraf adm
sudo -- bash -c 'echo "telegraf ALL=(ALL) NOPASSWD:ALL" >> /etc/sudoers'
```
You can check telegram service status:
```
sudo systemctl status telegraf
```
Status can be not ok with default Telegraf's config. Next steps will fix it.

Clone this project repo and copy variable script template
```
git clone https://github.com/svv28/mon_IXO.git
cd mon_IXO
cp mon_var_template.sh mon_var.sh
nano mon_var.sh
```

Insert your parameters to **mor_var.sh**:
* full path to ixo binary to COS_BIN_NAME ( check ```which ixod```)
* node PRC port to COS_PORT_RPC ( check in file ```path_to_ixo_node_config/config/config.toml```)
* node validator address to COS_VALOPER ( like ```ixovaloper********```)

Save changes in mon_var.sh and enable execution permissions:

```
chmod +x monitor.sh mon_var.sh
```

Edit telegraf configuration
```
sudo mv /etc/telegraf/telegraf.conf /etc/telegraf/telegraf.conf.orig
sudo nano /etc/telegraf/telegraf.conf
```
Copy it to config and paste your server name (to do so it is convenient to use the node moniker):
```
# Global Agent Configuration
[agent]
  hostname = "YOUR_MONIKER/SERVER_NAME" # set this to a name you want to identify your node in the grafana dashboard
  flush_interval = "15s"
  interval = "15s"
# Input Plugins
[[inputs.cpu]]
  percpu = true
  totalcpu = true
  collect_cpu_time = false
  report_active = false
[[inputs.disk]]
  ignore_fs = ["devtmpfs", "devfs"]
[[inputs.io]]
[[inputs.mem]]
[[inputs.net]]
[[inputs.system]]
[[inputs.swap]]
[[inputs.netstat]]
[[inputs.diskio]]
# Output Plugin InfluxDB
[[outputs.influxdb]]
  database = "ixometricsdb"
  urls = [ "MONITORING_SERV_URL:PORT" ] # example http://yourownmonitoringnode:8086
  username = "DB_USERNAME" # your database username
  password = "DB_PASSWORD" # your database user's password
[[inputs.exec]]
  commands = ["sudo su -c IXO_BIN_NAME -s /bin/bash IXO_USER"] # change home and username to the useraccount your validator runs at
  interval = "15s"
  timeout = "5s"
  data_format = "influx"
  data_type = "integer""
```

## Dashboard interface 

Dashboard has main cosmos-based node information and common system metrics. There is a description in it.

![Dashboard screenshort01](https://raw.githubusercontent.com/svv28/mon_IXO/main/resource/01_mon_evmos_grafana_dashboard.png "Dashboard screenshort01")
![Dashboard screenshort02](https://raw.githubusercontent.com/svv28/mon_IXO/main/resource/02_mon_evmos_grafana_dashboard.png "Dashboard screenshort02")

### Mon health
Complex parameter can show problem concerning receiving metrics from node. Normal value is "OK"

### Sync status
Node catching_up parameter

### Block height
Latest blockheight of node 

### Time since latest block
Time interval in seconds between taking the metric and node latest block time. Value greater 15s may indicate some kind of synchronization problem.

### Peers
Number of connected peers 

### Jailed status
Validator jailed status. 

### Missed blocks
Number of missed blocks in 10000 blocks running window. If the validator misses more than 500 blocks, it will end up in jail.

### Bonded status
Validator stake bonded info

### Voting power
Validator voting power. If the value of this parameter is zero, your node isn't in the active pool of validators 

### Delegated tokens
Number of delegated tokens

### Version
Version of ixod binary

### Vali Rank
Your node stake rank 

### Active validator numbers
Total number of active validators

### Other common system metrics: CPU/RAM/FS load, etc.
No comments needed)