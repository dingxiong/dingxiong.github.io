---
layout: post
title: How to write a Kafka consumer
date: 2023-12-08 00:01 -0800
categories: [kafka, consumer]
tags: [kafka, consumer]
---

Recently I have been writing the base library code to allow engineers to use
Kafka in my company easily. I followed some examples online to write a consumer
boilerplate code but quickly realized that there are so many pitfalls resulting
in inefficiency or even data loss.

After realizing how bad the consumer I wrote, I should find a good example from
Kafka's official team. OK. Kafka Connect is it! The `SinkConnector` dumps Kafka
messages to an external system. Check out
[WorkerSinkTask source code](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/WorkerSinkTask.java).

### A naive implementation

```python
while True:
    records = consumer.poll(1000)
    process(records)
    consumer.commit()
consumer.close()
```

This is the version I found online. It has many problems. Let's talk about them
one by one. The first problem is error handling. If `process(records)` throws
an exception, then the while loop stops and the consumer process stops. This
behavior is not what we want in production.

### Version 1

```python
while True:
  try:
      records = consumer.poll(1000)
      process(records)
  except Exception:
      pass
  finally:
      consumer.commit()
consumer.close()
```

This version handles errors gracefully. But it introduces another problem:
message loss. `process(records)` may throw exceptions in the middle, such that
some messages are processed, but the rest are not. The `finally` clause,
however, commits all messages.

### Version 2

```python
while True:
  try:
      records = consumer.poll(1000)
      for r in records:
        try:
            process_single_record(r)
        except BaseException:
            pass
  except BaseException:
      pass
  finally:
      consumer.commit()

consumer.close()
```

Note, here we capture `BaseException` because we need to capture system exit
exception too. But catching system exit exception sounds a lot. What if there
is a serious issue, and we really want to immediately exit. Another issue is
performance. There is no need to commit if there is no records.

### Version 3

```python
previous_offsets: dict[TopicParition, offset] = {}
curent_offsets: dict[TopicParition, offset] = {}

while True:
  try:
    records = consumer.poll(1000)
    for r in records:
      try:
          process_single_record(r)
          current_offset[r.topic_partion] = r.offset + 1
      except Exception:
          pass
  except Exception:
      pass
  Finally:
    if current_offset != previous_offsets:
      consumer.commit(curent_offsets);
      previous_offsets = current_offset
      current_offset.clear()
consumer.close()
```

A safer way is to manage offsets by ourselves. We only commit messages that we
have consumed. Also, it avoids unnecessary commits.

### Version 4

We can do better to handle system restart more gracefully. Instead of killing
consumer in the middle of processing. We can stop the processing loop after
finishing current iteration.

```python
previous_offsets: dict[TopicParition, offset] = {}
curent_offsets: dict[TopicParition, offset] = {}
should_stop = False

install_sig_hanlder() # such that when a SIGTERM signal comes, it sets should_stop = True

while not should_stop:
  try:
    records = consumer.poll(1000)
    for r in records:
      try:
          process_single_record(r)
          current_offset[r.topic_partion] = r.offset + 1
      except Exception:
          pass
  except Exception:
      pass
  Finally:
    if current_offset != previous_offsets:
      consumer.commit(curent_offsets);
      previous_offsets = current_offset
      current_offset.clear()
consumer.close()
```

### Version 5

The sequence of improvements above already makes the consumer logic quite
robust. We can improve the efficiency even more by reducing commit frequency.
Instead of committing after each batch, we can set a time interval threshold
for committing. But this design needs much more effort because we must remember
to do a final commit before current process exits. Also, we need to handle
dynamic rebalancing. I do not have a short pseudo-code for it. Check out
[WorkerSinkTask source code](https://github.com/apache/kafka/blob/1ae6405c479636bc0a4e0ffda91c82ea3bd3a761/connect/runtime/src/main/java/org/apache/kafka/connect/runtime/WorkerSinkTask.java)
instead.
