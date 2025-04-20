---
title: Kafka misc
date: 2023-12-08 09:37 -0800
categories: [kafka]
tags: [kafka]
---

A good material to checkout for kafka
https://jaceklaskowski.gitbooks.io/apache-kafka/content/kafka-controller.html

## Concepts:

- isr: An in-sync replica (ISR) is a broker that has the latest data for a
  given partition
- fence broker: broker failed heartbeat.

## replication factor vs number of brokers

Replication factor should never be larger than the number of brokers. See
[source code](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/metadata/src/main/java/org/apache/kafka/metadata/placement/StripedReplicaPlacer.java#L416-L416).

## Group coordinator vs Kafka Controller

Each consumer group has a coordinator. API `FindCoordinator` is used to find
group coordinator.

## Build

Checkout https://github.com/apache/kafka and

```bash
# you may need to use a lower version of Java
export JAVA_HOME=/Users/xiongding/Library/Java/JavaVirtualMachines/azul-13.0.14/Contents/Home

# do not running tests as it takes too much time.
./gradlew build -x test

# run a specific test
./gradlew clients:test --tests '*PemKeyStoreFileWithKeyPassword*'
```

## Code structure

### Auto generated files

Requests and responses that are specified in Kafka's binary protocols are auto
generated. These are json files under `resources` folder. If you do not build
the repo, you will see lots of import errors in IDE. For example:
[CreatePartitionsRequest](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/clients/src/main/resources/common/message/CreatePartitionsRequest.json).

An interesting thing is that these requests and responses schemas are defined
in `client` project. And `core` project
[depends on](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/build.gradle#L867-L867)
`client` projects, so `core` can use these definitions. I just feel this
structure is little weird.

### Service entry point

Kafka's binary protocols is built on top of TCP, namely stream sockets in
Unix-like stack, so it is still a request-response model. The handlers for
these endpoints are defined in two places:

- [ControllerApis.scala](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/server/ControllerApis.scala#L82-L82)
- [KafkaApis](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/server/KafkaApis.scala#L89-L89)
