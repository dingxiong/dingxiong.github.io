---
title: Kafka consumer groups
date: 2023-12-08 09:31 -0800
categories: [kafka, consumer]
tags: [kafka, consumer]
---

`kafka-consuemr-group.sh` is a script released together with Kafka that helps
users to inspect or change consumers.

The operations it supports are `ListGroups`, `DescribeGroups`, `DeleteGroups`,
`ResetOffsets` and `DeleteOffsets`. See source code
[here](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/admin/ConsumerGroupCommand.scala#L66-L66).
Let's talk about them one by one.

## Describe Groups

Example

```
$ /opt/homebrew/Cellar/kafka/3.4.0/bin/kafka-consumer-groups --bootstrap-server localhost:9092 --describe --all-groups
Consumer group 'group-1-INFRA-' has no active members.
GROUP           TOPIC                    PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
group-1-INFRA-  development-infra-test-1 0          1391            1391            0               -               -               -
Consumer group 'ip-group-1-INT-' has no active members.
GROUP           TOPIC                                  PARTITION  CURRENT-OFFSET  LOG-END-OFFSET  LAG             CONSUMER-ID     HOST            CLIENT-ID
ip-group-1-INT- development-integration_webhook_events 0          0               0               0               -               -               -
```

For each consumer group, it prints out the corresponding topic, partition,
current offset and end offset. The `LOG-END-OFFSET` is obtained from
`ListOffsets` API (ApiKey = 2). This API can fetch the offset at any given
timestamp. See
[code](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/clients/src/main/java/org/apache/kafka/clients/admin/OffsetSpec.java#L24-L24).
Most time, we only care about the earliest and latest offsets. These two
timestamps have special timestamp value
[-2 and -1](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/clients/src/main/java/org/apache/kafka/common/requests/ListOffsetsRequest.java#L41-L41).

This API allows us to get the offsets for a list of topic and partitions. You
may be attempted to fetch the earliest and latest offsets in one request, but
this does not work. The API checks if there are
[duplicated partitions](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/clients/src/main/java/org/apache/kafka/common/requests/ListOffsetsRequest.java#L103-L103)
in the request. So you need two requests to get the earliest and latest offsets
for a list of partitions.

This API has 8 versions in total. But I found that version 1 and above are
handled uniformly. See code
[here](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/server/KafkaApis.scala#L1002-L1002).
That's probably why `dpkp/kafka-python` only uses
[version 0 and 1](https://github.com/dpkp/kafka-python/blob/12325c09baefae2396f1083bc8b037603721198c/kafka/consumer/fetcher.py#L573)
to call this API. By the way, I failed to call this API using version 5.

## Reset Offsets

```
kafka-consumer-groups.sh --bootstrap-server <kafka_broker_host:9091> --group <group_name> --reset-offsets --to-latest --topic <my-topic> --execute
```
