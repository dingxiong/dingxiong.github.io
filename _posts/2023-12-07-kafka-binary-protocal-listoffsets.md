---
title: Kafka binary protocal -- ListOffsets
date: 2023-12-07 17:40 -0800
categories: [kafka, binary-protocol]
tags: [kafka, binary-protocol, kcat, listoffsets]
---

First question: what is the unit of `timestamp` inside Kafka repo. The
`Long timestamp` variable shows everywhere inside Kafka repo. What is its unit?
Take a step back. The timestamp must come from the producer side. See
[code](https://github.com/apache/kafka/blob/2.8.1/clients/src/main/java/org/apache/kafka/clients/producer/KafkaProducer.java#L940-L940).
It is the milliseconds since Unix epoch!

## ListOffsets
