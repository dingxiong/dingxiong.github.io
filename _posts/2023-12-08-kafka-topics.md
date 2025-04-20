---
title: Kafka topics
date: 2023-12-08 09:31 -0800
categories: [kafka]
tags: [kafka]
---

## Source code

Kafka provides a script
[kafka-topics.sh](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/bin/kafka-topics.sh#L17-L17)
to manage topics. It simple class
[TopicCommand.scala](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/admin/TopicCommand.scala).
As you can see it supports `create`, `alert`, `list`, `describe` and `delete`
commands.

## Change number of partitions

```
/bin/kafka-topics.sh --bootstrap-server localhost:9092 --alter --topic apache_event_log_topic --partitions 4
```

This commands calls
[CreatePartitions](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/admin/TopicCommand.scala#LL281C21-L281C37)
and the corresponding backend logic starts from
[here](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/core/src/main/scala/kafka/server/ControllerApis.scala#L107).
The core part is function
[createPartitions](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/metadata/src/main/java/org/apache/kafka/controller/ReplicationControlManager.java#L1482-L1482).
There are a few constraint for this api:

1. The input topic names should not have duplicates.
2. You must have write permission on the topic.
3. The new partition number must be larger than the current number. Note, I
   said larger. If it is equal, it will error out.

Existing partitions will not be modified. For the new partitions, how does
Kafka distribute them to brokers? This API request contract has `Assignments`
parameter which can be used to designate assignments. If this parameter is not
specified, then Kafka uses a round-robin way to do the assignment. Read
[StripedReplicaPlacer](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/metadata/src/main/java/org/apache/kafka/metadata/placement/StripedReplicaPlacer.java#L35-L35)
to learn more details.

## Delete topics

```
/bin/kafka-topics.sh --bootstrap-server localhost:9092 --delete --topic <topic1>,<topic2>
```

We can delete multiple topics in a single command. Simply separate the topics
using comma. This `TopicCommand` class will translate `,` to `|` and then do a
regex matching. See
[code](https://github.com/apache/kafka/blob/main/core/src/main/scala/kafka/utils/TopicFilter.scala#L28-L28).
Basically, we can use a regex if we want to.
