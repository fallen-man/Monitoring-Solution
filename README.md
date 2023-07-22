Cloud Monitoring System Design
=================
this document explaines how to monitor a acloud environment with observability, reliability, high availability in mind

there are many ways of doing this job done. like using prometheus with thanos components, graphite or influxdb.

i choose to using influxdb cluster, an `open-source alternative to influx enterprise` that let us create a cluster of influx `data` and `meta` nodes.
meta nodes are used for consensus and data nodes stores data in shards with specific retention.
we can have many databases with or without shards and replication factor

this approach will cover the scalability and high availability of data.

what is used is : https://github.com/chengshiwen/influxdb-cluster/tree/master
ofcourse we can use influx enterprise, and i need to mention that installation process of this alternative is just like influx enterprise.
so we need to configure at least three meta nodes because of RAFT consesnsus algorithm used by meta nodes. and at least two data nodes for HA.
hardware recommendation for `meta nodes` is : `16GB ram / 8 core CPU` and  for `meta nodes` is: `2GB ram / 1 core cpu` (meta nodes doesn't have heavy computations)
 
## i had not enough resources to bring up 5 nodes on VM's , so i used docker containers for creating this db cluster. but the production grade cluster must be configured on standalone OS.

firts we need to configure meta nodes:
https://github.com/chengshiwen/influxdb-cluster/wiki/Home-Eng#meta-node-setup

and the data nodes:
https://github.com/chengshiwen/influxdb-cluster/wiki/Home-Eng#data-node-setup


and then we can verify with "influx-ctl show"

data nodes will recieve data on port 8086 by default and we can use HA-Proxy to load balancing r/w requests on both data nodes, also we can use 2 HA-proxies with keepalived to implement vrrp for HA
of HA-proxy itself.

## i have to mention that we can run influxdb cluster using influxdb 1.6 and lower, but this verions are not supported anymore. also nodes can be hybrid too in old setup.

======================================================================================================================================================================================================

for feeding data to influxdb from our DCs i choose to go with telegraf thats a part of influxdata stack , telegraf can fetch data from multiple inputs and after adding metadatas like DC-name , rack-no and etc... feeds the data to multiple outputs like influxdb itself.

there is 2 key sections in telegarf configuration file :
[outputs.<provider_name>]
[inputs.<provider_name>]

there is a bunch of output and input providers that can be used for future use cases when we need to monitor other stuff like kernel metrics or docker containers and so on.
i used [inputs.libvirt] for monitor metrics of kvm/qemu vms and [outputs.influxdb] to feed the data to influxdb cluster.

you can install telegraf based on your distro with according to official documentation:
https://docs.influxdata.com/telegraf/v1.9/introduction/installation/

======================================================================================================================================================================================================

for visualizing our metrics there is many ways as many visualization tools can integrate with influxdb like chronograf and grafana.
as grafana is a popular one, so i picked it up for this project.

grafana can easily read time series data from influxdb and will support the integration, natively.
* i must mention that all the connections between grafana , telegraf and influxdb can be encrypted with TLS.

finally we can use grafana to query metrics from influxdb using influxQL and flux (flux will support in influxdb v1.8+)

also for sending alert to Discord channels, we can use grafana managed alerts and contanct point that supports discord out-of-the-box
======================================================================================================================================================================================================

++ how to install kafka ++

sudo apt update && sudo apt install default-jre

sudo adduser kafka
sudo adduser kafka sudo
su -l kafka

mkdir ~/Downloads
curl "https://downloads.apache.org/kafka/3.5.0/kafka_2.13-3.5.0.tgz" -o ~/Downloads/kafka.tgz
mkdir ~/kafka && cd ~/kafka
tar -xvzf ~/Downloads/kafka.tgz --strip 1

# till this part, we downloaded and decompressed kafka, keep in mind that if you download kafka source, you need to make it with gradle first.
vim ~/kafka/config/server.properties

sudo nano /etc/systemd/system/zookeeper.service : 

[Unit]
Requires=network.target remote-fs.target
After=network.target remote-fs.target

[Service]
Type=simple
User=kafka
ExecStart=/home/kafka/kafka/bin/zookeeper-server-start.sh /home/kafka/kafka/config/zookeeper.properties
ExecStop=/home/kafka/kafka/bin/zookeeper-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

sudo nano /etc/systemd/system/kafka.service : 
[Unit]
Requires=zookeeper.service
After=zookeeper.service

[Service]
Type=simple
User=kafka
ExecStart=/bin/sh -c '/home/kafka/kafka/bin/kafka-server-start.sh /home/kafka/kafka/config/server.properties > /home/kafka/kafka/kafka.log 2>&1'
ExecStop=/home/kafka/kafka/bin/kafka-server-stop.sh
Restart=on-abnormal

[Install]
WantedBy=multi-user.target

sudo systemctl start kafka
sudo systemctl status kafka
sudo systemctl enable zookeeper
sudo systemctl enable kafka


** we need to configure zookeeper on every node first
zookeeper config for clustering:

tickTime=2000
dataDir=/var/zookeeper/
clientPort=2181
initLimit=5
syncLimit=2
server.1=<IP>:2888:3888
server.2=<IP>:2888:3888
server.3=<IP>:2888:3888

** also we need to create a file name myid in dataDir of zookeeper. the file must be contain a number between 1 to 255 and nothing else. first node can be just 1 and for our project 2 and 3.



broker.id=1 listeners=PLAINTEXT://localhost:9092 num.partitions=3 log.dirs=/tmp/kafka-logs-1 zookeeper.connect=localhost:2181 broker.id=2 listeners=PLAINTEXT://localhost:9093 num.partitions=3 log.dirs=/tmp/kafka-logs-2 zookeeper.connect=localhost:2181 broker.id=3 listeners=PLAINTEXT://localhost:9094 num.partitions=3 log.dirs=/tmp/kafka-logs-3 zookeeper.connect=localhost:2181 