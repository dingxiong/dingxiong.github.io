---
title: Kafka operations
date: 2023-12-07 23:52 -0800
categories: [kafka, misc]
tags: [kafka, kcat]
---

## list topics

```
./bin/kafka-topics.sh --bootstrap-server localhost:9092 --list
./bin/kafka-topics.sh --describe --bootstrap-server localhost:9092 --topic filebeat
./bin/kafka-run-class.sh kafka.tools.GetOffsetShell --broker-list localhost:9092 --topic filebeat --time -1
./bin/kafka-consumer-offset-checker.sh --zookeeper=kafka-main-zookeeper-client.kafka:2181 --topic=filebeat --group=logstash
```

## show configs

```
./bin/kafka-topics.sh --version
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type topics --entity-name filebeat --describe --all
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-default --describe
./bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-name 0 --describe

kafkacat -L -b $BOOTSTRAP_SERVERS
```

## show offsets

```
./bin/kafka-consumer-groups.sh --bootstrap-server localhost:9092 --describe --all-groups
./bin/kafka-consumer-groups.sh --describe --group logstash --bootstrap-server localhost:9092

```

## Reassign partition

```
[kafka@kafka-main-kafka-2 kafka]$ cat /home/kafka/reassignment.json
{
  "partitions": [
    {
      "topic": "filebeat",
      "partition": 0,
      "replicas": [
        1,
        0
      ]
    },
    {
      "topic": "filebeat",
      "partition": 1,
      "replicas": [
        2,
        1
      ]
    },
    {
      "topic": "filebeat",
      "partition": 2,
      "replicas": [
        0,
        2
      ]
    },
    {
      "topic": "filebeat",
      "partition": 3,
      "replicas": [
        1,
        2
      ]
    },
    {
      "topic": "filebeat",
      "partition": 4,
      "replicas": [
        2,
        0
      ]
    },
    {
      "topic": "filebeat",
      "partition": 5,
      "replicas": [
        0,
        1
      ]
    }
  ]
}

./bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --generate --broker-list 0,1,2 --topics-to-move-json-file /home/kafka/topics.json
./bin/kafka-reassign-partitions.sh --bootstrap-server localhost:9092 --execute --reassignment-json-file /home/kafka/reassignment.json
```

## run a kafka consumer

```

./bin/kafka-console-consumer.sh --bootstrap-server localhost:9092 --topic
filebeat

k run kafka-consumer -n kafka -ti
--image=quay.io/strimzi/kafka:latest-kafka-3.3.1 --rm=true --restart=Never --
bin/kafka-console-consumer.sh --bootstrap-server logs-kafka-bootstrap:9092
--topic filebeat --from-beginning

k run kafka-consumer -n staging-oneoff
--image=quay.io/strimzi/kafka:latest-kafka-2.8.1 --rm=true --restart=Never -it
\
--overrides='{"spec":{"nodeSelector":{"role":"staging-job"},"tolerations":[{"effect":
"NoSchedule","key": "dedicated","operator": "Equal","value": "staging-job"}]}}'
\
-- bash

k run kafka-consumer -n staging-oneoff --image=bitnami/kafka:2.8.1 --rm=true
--restart=Never -it \
--overrides='{"spec":{"nodeSelector":{"role":"staging-job"},"tolerations":[{"effect":
"NoSchedule","key": "dedicated","operator": "Equal","value": "staging-job"}]}}'
\
-- bash

```

## run a kafka producer

```

k run kafka-producer -ti --image=quay.io/strimzi/kafka:0.32.0-kafka-3.3.1
--rm=true --restart=Never -- bin/kafka-console-producer.sh --bootstrap-server
logs-kafka-bootstrap.kafka:9092 --topic filebeat

```

## Debug network issue

Sometimes, you have a valid bootstrap server, but the metadata returned by
Kafka broker is unexpected.

```

# step 1: install java 8 or 11

# step 2: download kafka

rm -rf /tmp/debug mkdir -p /tmp/debug curl
https://archive.apache.org/dist/kafka/2.8.1/kafka_2.12-2.8.1.tgz -o
/tmp/debug/kafka.tgz cd /tmp/debug tar -xvzf kafka.tgz --strip 1

# step 3: turn on debug logging

cat > tools-log4j.properties <<EOF log4j.rootLogger=DEBUG, stderr

log4j.appender.stderr=org.apache.log4j.ConsoleAppender
log4j.appender.stderr.layout=org.apache.log4j.PatternLayout
log4j.appender.stderr.layout.ConversionPattern=[%d] %p %m (%c)%n
log4j.appender.stderr.Target=System.err EOF

export
KAFKA_OPTS="-Dlog4j.configuration=file:/tmp/debug/tools-log4j.properties"

# step 4: call the config description endpoint.

# The debug log will show what metadata response look like

./bin/kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers
--all --describe

```

Most time, it is the issue of `listeners` and `advertised.listeners`.

# Resources

- https://snourian.com/kafka-kubernetes-strimzi-part-1-creating-deploying-strimzi-kafka/

```

```
