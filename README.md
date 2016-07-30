uMirrorMaker 
============
uMirrorMaker provides the ability to replicate across Kafka clusters in other data centers. Instead of publishing to a single Kafka cluster, you can publish data to multiple regional Kafka clusters and aggregate it all in one Kafka cluster.

=========

Kafka's current (part of 0.8.2) MirrorMaker design consumes data from multiple regional Kafka clusters using a Kafka high-level consumer. With this design, any regional Kafka broker experiencing issues causes rebalancing across all of the active topics.

uMirrorMaker introduces a Helix-based controller and uses a simple consumer in MirrorMaker Worker. This design achieves the following goals:

1. Stability: Rebalance only occurs during startup (when a node is added/deleted)
2. Simple operations: Easy to scale up cluster, no server restart for whitelisting topics
3. High throughput: Max offset lag is consistently 0.
4. Time SLA (~5min)

# uMirrorMaker Quick Start

## Get the Code
Check out the uMirrorMaker project:
```
git clone git@github.com:uber/umirrormaker.git
cd umirrormaker
```
This project contains everything (both mirrormaker-controller and mirrormaker-worker) you’ll need to run uMirrorMaker.

## Build uMirrorMaker
Before you can run uMirrorMaker, you need to build a package for it. This package is what your deployment tool uses to deploy uMirrorMaker.
```
mvn clean package
```
Or command below (the previous one will take a long time to run):
```
mvn clean package -DskipTests
```

## Set Up Local Test Environment
To test uMirrorMaker locally, you need two systems: [Kafka](http://kafka.apache.org/), and [ZooKeeper](http://zookeeper.apache.org/). The script “grid” is to help you set up these systems.
- Modify permission for the scripts generated by Maven:
```
chmod u+x bin/pkg/*.sh
```
- The command below will download, install, and start ZooKeeper and Kafka (will start two Kafka systems: kafka1, which we use as source Kafka cluster, and kafka2, which we use as destination Kafka cluster):
```
bin/grid bootstrap
```
- Create a dummyTopic in kafka1 and produce some dummy data:
```
./bin/produce-data-to-kafka-topic-dummyTopic.sh
```
- Check if the data is successfully produced to kafka1 by opening another console tab and executing the command below:
```
./deploy/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181/cluster1 --topic dummyTopic
```
- You should get this data:
```
Kafka topic dummy topic data 1
Kafka topic dummy topic data 2
Kafka topic dummy topic data 3
Kafka topic dummy topic data 4
…
```

## Start uMirrorMaker

**Example 1**: Copy data from source cluster to destination cluster

- Start uMirrorMaker Controller (you should keep it running):
```
./bin/pkg/start-controller-example1.sh
```

- Start uMirrorMaker Worker (you should keep it running, and it’s normal if you see kafka.consumer.ConsumerTimeoutException at this moment, since no topic has been added for copying):
```
./bin/pkg/start-worker-example1.sh
```

- Add topic to uMirrorMaker Controller to start copying from kafka1 to kafka2:
```
curl -X POST -d '{"topic":"dummyTopic", "numPartitions":"1"}' http://localhost:9000/topics
```

- To check if the data is successfully copied to kafka2, you should open another console tab and execute the command below:
```
./deploy/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181/cluster2 --topic dummyTopic1
```

- And you will see the same messages produced in kafka1:
```
Kafka topic dummy topic data 1
Kafka topic dummy topic data 2
Kafka topic dummy topic data 3
Kafka topic dummy topic data 4
…
```

**Example 2**: Copy data from source Kafka to destination Kafka cluster without explicitly whitelisting topics

- Start uMirrorMaker Controller (you should keep it running):
```
./bin/pkg/start-controller-example2.sh
```

- Start uMirrorMaker Worker (you should keep it running, and it’s normal if you see kafka.consumer.ConsumerTimeoutException at this moment since no topic has been added for copying):
```
./bin/pkg/start-worker-example2.sh
```

- Create topic in kafka2. Example 2 enables topic auto-whitelisting, so you don't need to whitelist topics manually. If a topic is in both source and destination Kafka clusters, the controller auto-whitelists the topic and starts copying data.

```
./deploy/kafka/bin/kafka-topics.sh --create --zookeeper localhost:2181/cluster2 --topic dummyTopic --partition 1 --replication-factor 1
```

- To check if the data is successfully copied to kafka2, open another console tab and execute this command (you might need to wait about 20 seconds for controller to refresh):
```
./deploy/kafka/bin/kafka-console-consumer.sh --zookeeper localhost:2181/cluster2 --topic dummyTopic
```

- And you should see the same messages produced in kafka1:
```
Kafka topic dummy topic data 1
Kafka topic dummy topic data 2
Kafka topic dummy topic data 3
Kafka topic dummy topic data 4
…
```

## Shutdown
When you’re done, you can clean everything up using the same grid script:
```
./bin/pkg/stop-all.sh
```

Congratulations! You’ve now set up a local grid that includes Kafka and ZooKeeper, and you've run a uMirrorMaker worker on it.
