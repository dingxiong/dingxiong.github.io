---
layout: post
title: Kafka configurations
date: 2023-12-08 09:31 -0800
categories: [kafka]
tags: [kafka]
---

There are tons of parameters to configure Kafka. The configuration can be
classified into 3 buckets: broker configurations, producer configurations and
consumer configurations.

## Broker configuration dynamic updates

Usually, broker starts with configurations defined inside `server.properties`,
anytime we need to update the configuration, broker needs to be restarted. This
is very inefficient. So starting from Kafka 1.1.0, some broker configurations
can be dynamically updated without broker reboot. See
[KAFKA-6240](https://issues.apache.org/jira/browse/KAFKA-6240) for more
details. You can check
[this table](https://kafka.apache.org/11/documentation.html#brokerconfigs) to
see which configurations can be updated dynamically.

## Message size

By default, the largest message Kafka can support by default is 1MB. If you
want to increase this threshold, what parameter need to change?
[This post](https://www.conduktor.io/kafka/how-to-send-large-messages-in-apache-kafka/)
provides a good overview.

On broker side, we need to change two parameters `message.max.bytes` and
`replica.fetch.max.bytes`. The former can be changed dynamically, but the
latter requires broker restart.

On producer side, `max.request.size` and `buffer.memory` needs to change.
Otherwise, you will see a
[RecordTooLargeException](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L1176).

On consumer side, `max.partition.fetch.bytes` needs to be changed.
