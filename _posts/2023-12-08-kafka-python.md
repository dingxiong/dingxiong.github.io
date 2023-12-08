---
title: Kafka Python
date: 2023-12-08 09:31 -0800
categories: [kafka, mics]
tags: [kafka, python]
---

## Kafka python client

There are two popular python client library:
[kafka-python](https://github.com/dpkp/kafka-python) and
`confluent-python-kafka`. The former is a python translation of the official
Java client.

## dpkp/kafka-python

```python
# How to get Server supported API versions
admin_client._client.get_api_versions()


```
