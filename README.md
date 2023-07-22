Cloud Monitoring System Design
=================
this document explaines how to monitor a acloud environment with observability, reliability, high availability in mind

there are many ways of doing this job done. like using prometheus with thanos components, graphite or influxdb.

## influxdb cluster

i choose to using influxdb cluster, an `open-source alternative to influx enterprise` that let us create a cluster of influx `data` and `meta` nodes.
meta nodes are used for consensus and data nodes stores data in shards with specific retention.
we can have many databases with or without shards and replication factor

this approach will cover the scalability and high availability of data.

what is used is : https://github.com/chengshiwen/influxdb-cluster/tree/master
ofcourse we can use influx enterprise, and i need to mention that installation process of this alternative is just like influx enterprise.
so we need to configure at least three meta nodes because of RAFT consesnsus algorithm used by meta nodes. and at least two data nodes for HA.
hardware recommendation for `meta nodes` is : `16GB ram / 8 core CPU` and  for `meta nodes` is: `2GB ram / 1 core cpu` (meta nodes doesn't have heavy computations)
 
`## i had not enough resources to bring up 5 nodes on VM's , so i used docker containers for creating this db cluster. but the production grade cluster must be configured on standalone OS.`

firts we need to configure meta nodes: [Meta-Node-Setup-Guid](https://github.com/chengshiwen/influxdb-cluster/wiki/Home-Eng#meta-node-setup)
and the data nodes: [Data-Node-Setup-Guide](https://github.com/chengshiwen/influxdb-cluster/wiki/Home-Eng#data-node-setup)

after this steps, verify your cluster using `influx-ctl show`

data nodes will recieve data on port 8086 by default and we can use HA-Proxy to load balancing r/w requests on both data nodes, also we can use 2 HA-proxies with keepalived to implement vrrp for HA
of HA-proxy itself.

 `i have to mention that we can run influxdb cluster using influxdb 1.6 and lower, but this verions are not supported anymore. also nodes can be hybrid too in old setup.`

## Telegraf

for feeding data to influxdb from our DCs i choose to go with telegraf thats a part of influxdata stack , telegraf can fetch data from multiple inputs and after adding metadatas like DC-name , rack-no and etc... feeds the data to multiple outputs like influxdb itself.

there is 2 key sections in telegarf configuration file :
`[outputs.<provider_name>]`
`[inputs.<provider_name>]`

there is a bunch of output and input providers that can be used for future use cases when we need to monitor other stuff like kernel metrics or docker containers and more.
i used `[inputs.libvirt]` for monitor metrics of kvm/qemu vms and `[outputs.influxdb]` to feed the data to influxdb cluster, `[outputs.kafka]` for sending metrics to kafka topics and `[inputs.kafka_consumer]` for consume metrics from kafka topics and feed them to influxdb.

i just added kafka cluster in between of 2 telegraf instances , one on the kvm-running node to collecting metrics, add tag to them and forward metrics to kafka topics. and another standalone telegraf instance that consumes data from kafka cluster and writes them to influxdb.
* in consumer telegraf instance, we can add all our kafka nodes to consume for high-availability

you can install telegraf based on your distro with according to official [documentation](https://docs.influxdata.com/telegraf/v1.9/introduction/installation/)

## Grafana

for visualizing our metrics there is many ways as many visualization tools can integrate with influxdb like chronograf and grafana.
as grafana is a popular one, so i picked it up for this project.

grafana can easily read time series data from influxdb and will support the integration, natively.

`i must mention that all the connections between grafana , telegraf and influxdb can be encrypted with TLS.`

finally we can use grafana to query metrics from influxdb using influxQL and flux (flux will support in influxdb v1.8+)

also for sending alert to Discord channels, we can use grafana managed alerts and contanct point that supports discord out-of-the-box
## Kafka

to have a more reliable and fail-tolerant system , i used kafka cluster with 3 nodes using zookeeper to implementing cluster. i must mention that in lastest kafka releases you can also use Kraft for clustering instead of zookeeper.

this setup can help us to send metrics for new services in future to new kafka topics, or using telegraf in consumer side on a k8s cluster or a docker-swarm as this consumer is stateless.
we can also use consumer-groups in this scenario.

to setup a kafka cluster, simply we need to downlaod kafka binary and configure our `/kafka/config/server.properties` and `/kafka/config/zookeeper.properties`.
* important tip here is we need to create a file named `myid` in zookeeper's `daraDir`. this file must contain just a number between 0 and 255 and must be unique for each node.
* another important tip is that in `zookeeper.properties` file when we defining servers, `server.<ID>:8222:8333` the `ID` segment for each node must be equals to myid file content.
* it's better to have a `broker.id=0` for `myid=1` and so on.
* we can configure retention policy for kafka topics as kafka can persist data streams, it's have a default of 168 hours and can be defined by `log.retention.hours=<time in hours>`
* as i mentioned before, you can consume a topic with telegraf from multiple kafka nodes and when a node goes down, telegram will automatically consume from another node.
* also you can config your producers to write to multiple kafka nodes that provides failover, but we can use load-balancers too; for a round-robin or leas-connect load balancing between kafka nodes.

## test the setup

for test this setup i've just used my ubuntu local machine to run vm's directly with KVM.
just installed telegraf on my machine with `[inputs.libvirt]` and `[outputs.kafka]` and tagged the metrics with `[global_tags]` and setup `key = value` specific tags like `dc = teh-1`.

it's important to know in this setup we can also tag the metrics with kafka producer and set kafka topic name that metric recieved from. this way we can trace our metrics and provide a better observability over our environment.

## Conclusion

there is many ways for monitoring a cloud environment an it can be picked based on our specific needs. so this document can not be the best way for every cloud environments.

you can see the topology of this architecture in extra folder of this repository.

## A Better Way
There is also a more fault-tolerant and reliable way to do this scenario:
we can distribute kafka cluster on our datacenters which every DC can have 3 kafka nodes including HQ DC.
this way, producing and consuming metrics will be faster and more reliable.