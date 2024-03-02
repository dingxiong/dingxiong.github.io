---
title: Kafka connect
date: 2023-12-08 00:00 -0800
categories:
categories: [kafka, connect]
tags: [kafka, kafka-connect]
---

# Worker vs Connector vs Task

When I read the Kafka Connect source code, the relationship between worker,
connector and task confused me.
[Confluent](https://docs.confluent.io/platform/current/connect/index.html#distributed-workers)
has a nice illustration about their relationship. To put it simple, worker is a
process that run task threads. It manages the life cycles of these tasks. A
connector consists multiple tasks. Each task is a thread. Based on whether it
produces messages to Kafka or consumes messages from Kafka, tasks are
classified into SourceTask and SinkTask. Usually these tasks are partitioned
according to the Kafka partition. I think the purpose of having multiple tasks
instead of one in a connector is to improve throughput. From
[source code](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/Worker.java#L607-L607),
you can see how worker starts and stops tasks.

We now understand that worker is container process, and tasks are threads
running in a worker process. How are they configured? First, all tasks
belonging to the same connector share the same configurations, so we only need
to talk about worker connectconfigs and connector configurations. Worker
configuration class is
[WorkerConfig](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/WorkerConfig.java#L53-L53).
Connector configurations are
[SinkConnectorConfig](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/SinkConnectorConfig.java#L37-L37)
and
[SourceConnectorConfig](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/SourceConnectorConfig.java#L39-L39).

## key.converter and value.converter

Both worker and connector can specify `key.converter` and `value.converter`,
and connector config has higher precedence. See
[code](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/Worker.java#L532-L532).

How does converter work?

Worker is responsible for (de)serialize message and task only need to deal with
object `SinkRecord` and `SourceRecord`. Both records are subtype of
`ConnectRecord` which has below fields:

```
private final Schema keySchema;
private final Object key;
private final Schema valueSchema;
private final Object value;
```

### JsonConverter

The
[JsonConverter](https://github.com/apache/kafka/blob/2.8.1/connect/json/src/main/java/org/apache/kafka/connect/json/JsonConverter.java#L64-L64)
expect the Kafka message can de decoded as a Json. A sample task configuration
is as below.

```
value.converter=org.apache.kafka.connect.json.JsonConverter
value.converter.schemas.enable=true
key.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
```

First, you notice that we use `key.converter` and `value.converter` to specify
the type of converter. Then, `key.converter.` and `value.converter.` are used
as prefix to assign properties of the corresponding converter. This is a common
trick inside Kafka and is used in many places. For this specific case, the
source code is
[here](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/isolation/Plugins.java#L279-L279).

What does config `schema.enable` do?

If this config is true, then the expected json message should has structure
`{"schema":..., "payload": ...}`. The `payload` part is the Kafka message. If
this config is false, then the whole message is used as `payload`. This is very
important when you use json converter for SinkTask. If you cannot guarantee the
Kafka message has as `payload` field, then set this config to false.

## producer and consumer configurations

Worker can specify Kafka producer and consumer configurations. It needs to add
a prefix `producer.` or `consumer.` to the properties. See
[code](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/Worker.java#L697-L697).
So all tasks running inside this worker will inherit these configurations.
Meanwhile, connector can overwrite these configurations by providing
configurations with prefix `producer.overwrite.` and `consumer.overwrite.`. See
[code](https://github.com/apache/kafka/blob/2.8.1/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/Worker.java#L702-L702).
However, this overwrite behavior is disabled by default. We need to allow
overwriting by setting

```
connector.client.config.override.policy=All
```

However, for MSK connect, this option is not supported. See
[AWS doc](https://docs.aws.amazon.com/msk/latest/developerguide/msk-connect-workers.html).

in worker configurations.

## config provider

# Load plugin

AWS MSK connect requires us to package Kafka connect plugin as a zip file and
load to S3. The it will build a plugin using this S3 file. This is super magic!
The immediate question raised in my mind is what structure this zip file should
have. Quote from
[Kafka official document](https://kafka.apache.org/documentation/#connectconfigs_plugin.path):

```
List of paths separated by commas (,) that contain plugins (connectors, converters, transformations). The list should consist of top level directories that include any combination of:
a) directories immediately containing jars with plugins and their dependencies
b) uber-jars with plugins and their dependencies
c) directories immediately containing the package directory structure of classes of plugins and their dependencies
Note: symlinks will be followed to discover dependencies or plugins.
```

It seems we should just need to put all jars files in a directory and this
directory is under on off `plugin.path`. It sounds quite confusing, right?

# Connect topics

Kafka will create 3 topics for each connector. In MSK, they are

```
__amazon_msk_connect_configs_database-1-binlogs-v8_0bce1b02-99f5-4f9d-8a38-d15ee190ad24-2,
__amazon_msk_connect_offsets_database-1-binlogs-v8_0bce1b02-99f5-4f9d-8a38-d15ee190ad24-2,
__amazon_msk_connect_status_database-1-binlogs-v8_0bce1b02-99f5-4f9d-8a38-d15ee190ad24-2,
```

## Configs topic

This topic is configured by
[config.storage.topic](https://kafka.apache.org/documentation/#connectconfigs_config.storage.topic).
Sample message inside this topic is

```
{
  "properties": {
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "topic.creation.default.partitions": "6",
    "tasks.max": "1",
    "schema.history.internal.consumer.sasl.jaas.config": "software.amazon.msk.auth.iam.IAMLoginModule required;",
    "transforms": "Reroute",
    "include.schema.changes": "true",
    "tombstones.on.delete": "false",
    "topic.prefix": "database-1",
    "transforms.Reroute.topic.replacement": "database-1.admincoin.mutations",
    "schema.history.internal.kafka.topic": "database-1-schema-change",
    "schema.history.internal.producer.security.protocol": "SASL_SSL",
    "topic.creation.default.replication.factor": "3",
    "value.converter": "org.apache.kafka.connect.json.JsonConverter",
    "key.converter": "org.apache.kafka.connect.json.JsonConverter",
    "schema.history.internal.producer.sasl.mechanism": "AWS_MSK_IAM",
    "schema.history.internal.consumer.sasl.client.callback.handler.class": "software.amazon.msk.auth.iam.IAMClientCallbackHandler",
    "database.user": "${secretManager:rds-staging:username}",
    "topic.creation.default.compression.type": "lz4",
    "transforms.Reroute.type": "io.debezium.transforms.ByLogicalTableRouter",
    "topic.creation.default.cleanup.policy": "delete",
    "database.server.id": "110",
    "schema.history.internal.producer.sasl.client.callback.handler.class": "software.amazon.msk.auth.iam.IAMClientCallbackHandler",
    "schema.history.internal.kafka.bootstrap.servers": "b-3.staging2.68am5b.c2.kafka.us-east-2.amazonaws.com:9098,b-1.staging2.68am5b.c2.kafka.us-east-2.amazonaws.com:9098,b-2.staging2.68am5b.c2.kafka.us-east-2.amazonaws.com:9098",
    "transforms.Reroute.topic.regex": "(.*)",
    "database.port": "3306",
    "key.converter.schemas.enable": "true",
    "database.hostname": "database-1.c0gaj5h9ndhp.us-east-2.rds.amazonaws.com",
    "database.password": "${secretManager:rds-staging:password}",
    "value.converter.schemas.enable": "true",
    "schema.history.internal.producer.sasl.jaas.config": "software.amazon.msk.auth.iam.IAMLoginModule required;",
    "schema.history.internal.consumer.sasl.mechanism": "AWS_MSK_IAM",
    "max.batch.size": "100",
    "database.include.list": "admincoin",
    "snapshot.mode": "schema_only",
    "schema.history.internal.consumer.security.protocol": "SASL_SSL",
    "name": "database-1-binlogs-v8"
  }
}
```

## Status topic

Sample message

```
{
  "state": "RUNNING",
  "trace": null,
  "worker_id": "172.21.74.63:8083",
  "generation": 3
}
```

## Offsets topic

Sample message

```
{
  "transaction_id": null,
  "ts_sec": 1693168282,
  "file": "mysql-bin-changelog.345610",
  "pos": 86250,
  "row": 1,
  "server_id": 1174305955,
  "event": 2
}
```

This topic answers my question what happens when connector crashed. Connector
should load the last offset from this topic and continue from there. See more
details about how it is used inside Debezium.

## Transformers

## Mics

### Useful logs

```
[Worker-02f381a61f0ac581c] [2023-09-01 06:22:20,097] INFO [database-1-binlogs-v9|task-0] using binlog 'mysql-bin-changelog.346991' at position '4250' and gtid '' (io.debezium.connector.mysql.MySqlSnapshotChangeEventSource:280)

INFO: Connected to evergreen-production-1.c0gaj5h9ndhp.us-east-2.rds.amazonaws.com:3306 at mysql-bin-changelog.391965/10661763 (sid:20, cid:85570953)
```
