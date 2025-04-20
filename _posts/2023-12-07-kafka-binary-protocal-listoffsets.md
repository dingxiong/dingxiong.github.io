---
title: Kafka binary protocal -- ListOffsets
date: 2023-12-07 17:40 -0800
categories: [kafka]
tags: [kafka, binary-protocol, kcat, listoffsets]
---

First question: what is the unit of `timestamp` inside Kafka repo. The
`Long timestamp` variable shows everywhere inside Kafka repo. What is its unit?
Take a step back. The timestamp must come from the producer side. See
[code](https://github.com/apache/kafka/blob/2.8.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L940-L940).
It is the milliseconds since Unix epoch!

## ListOffsets

The
[ListOffsets](https://kafka.apache.org/protocol.html#The_Messages_ListOffsets)
takes a list of topics and their partitions as arguments. The partition struct
takes a timestamp field meaning that you can get the offset corresponding at
that timestamp. The relevant code is
[here](https://github.com/apache/kafka/blob/2.8.1/core/src/main/scala/kafka/log/Log.scala#L1740-L1740).
This is useful for replaying traffic for backfilling purpose.

Note, this function returns the offset of the first message whose timestamp is
greater than or equals to the given timestamp. None if no such message is
found. So if you provide a timestamp of tomorrow, then it return None. Also, if
input timestamp is `-2`, then return the earliest offset. If input timestamp is
`-1`, then return the largest offset.

## Kafka library support

The official command line support get offsets as below.

```
bin/kafka-run-class.sh kafka.tools.GetOffsetShell --bootstrap-server <broker_list> --topic <topic_name> --time <timestamp>
```

However, it does not support providing SASL credentials, so it is nearly
useless in practice.

On the other hand, `kcat` does not have such constraint.

```bash
# timestamp older than the retention period, then earliest offset is returned
$ kafkacat -b ${BOOTSTRAP_SERVERS} -Q -t database-1.admincoin.mutations:0:1702020942826
database-1.admincoin.mutations [0] offset 4379021

# timestamp newer than the current time, then -1, i.e., None is returned
$ kafkacat -b ${BOOTSTRAP_SERVERS} -Q -t database-1.admincoin.mutations:0:1702120942826
database-1.admincoin.mutations [0] offset -1

# earliest offset
$ kafkacat -b ${BOOTSTRAP_SERVERS} -Q -t database-1.admincoin.mutations:0:-2
database-1.admincoin.mutations [0] offset 4378926

# largest offset
$ kafkacat -b ${BOOTSTRAP_SERVERS} -Q -t database-1.admincoin.mutations:0:-1
database-1.admincoin.mutations [0] offset 4739638
```

The python client `kafka-python` is stricter about the behavior.

```
In [39]: consumer.offsets_for_times({tp: 1702020912826})
Out[39]: {TopicPartition(topic='database-1.admincoin.mutations', partition=0): OffsetAndTimestamp(offset=4739434, timestamp=1702020914914)}

In [40]: consumer.offsets_for_times({tp: 1702020952826})
Out[40]: {TopicPartition(topic='database-1.admincoin.mutations', partition=0): None}

In [46]: consumer.beginning_offsets([tp])
Out[46]: {TopicPartition(topic='database-1.admincoin.mutations', partition=0): 4378926}

In [47]: consumer.end_offsets([tp])
Out[47]: {TopicPartition(topic='database-1.admincoin.mutations', partition=0): 4739637}
```

Note, python client does not support passing `-2` and `-1` to the timestamp
field, you need to use `beginning_offsets` and `end_offsets` instead.
