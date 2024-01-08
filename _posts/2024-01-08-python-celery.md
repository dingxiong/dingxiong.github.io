---
title: python celery
date: 2024-01-08 10:45 -0800
categories: [programming-language, python]
tags: [celery]
---

Celery has a AMQP broker and an consumer backend.

## kombu

Kombu is a messaging library that supports AMQP, redis, etc. It is basically an
AMQP broker. I feel reading its
[user guide](https://docs.celeryq.dev/projects/kombu/en/stable/userguide/index.html)
is super helpful to understand basic concepts.

Redis transport saves messages to a list. Redis key is just queue name or queue
name + priority.

- LPUSH to put message to queue.
  [code](https://github.com/celery/kombu/blob/f182a98e75e448b21fb2c12e69fb9565916ea32d/kombu/transport/redis.py#L1010)
- BRPOP to consume message at end of the queue.
  [code](https://github.com/celery/kombu/blob/f182a98e75e448b21fb2c12e69fb9565916ea32d/kombu/transport/redis.py#L983)

Use redis-cli monitor command, we can see a few examples.

> `1668301595.148637 [0 127.0.0.6:38131] "LPUSH" "payment_payout_jobs" "{\"body\": \"W1siYzdkOTdmNTUtYjI2MC00OGFkLTk2N2UtNDE5ZmM4YTBlYjRhIl0sIHt9LCB7ImNhbGxiYWNrcyI6IG51bGwsICJlcnJiYWNrcyI6IG51bGwsICJjaGFpbiI6IG51bGwsICJjaG9yZCI6IG51bGx9XQ==\", \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\", \"headers\": {\"lang\": \"py\", \"task\": \"jobs.payout.check_balance_and_trigger_payouts.log_bill_payouts_pending_zip_admin_actions_for_organization\", \"id\": \"42b870ea-acb6-4b17-8b63-08dddb86f9f2\", \"shadow\": null, \"eta\": null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\": 0, \"timelimit\": [null, 900.0], \"root_id\": \"1dcdc53b-8f4a-4df0-972a-dcdc262de100\", \"parent_id\": \"f45fb3dc-cc42-4178-9117-3c248f6e5570\", \"argsrepr\": \"('c7d97f55-b260-48ad-967e-419fc8a0eb4a',)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen122@celery-payment-payout-jobs-dd745d4f5-ktchv\", \"ignore_result\": false}, \"properties\": {\"correlation_id\": \"42b870ea-acb6-4b17-8b63-08dddb86f9f2\", \"reply_to\": \"a9c747ec-9849-30e4-a99f-1f039af363bc\", \"pre_enqueue_timestamp\": \"2022-11-13T01:06:35.147229\", \"delivery_mode\": 2, \"delivery_info\": {\"exchange\": \"\", \"routing_key\": \"payment_payout_jobs\"}, \"priority\": 0, \"body_encoding\": \"base64\", \"delivery_tag\": \"30b8b8af-f640-4bc9-a619-c86437b904d6\"}}"`

```
1668301595.150070 [0 127.0.0.6:58085] "BRPOP" "erp_core_data_sync_jobs" "erp_core_data_sync_jobs\x06\x163" "erp_core_data_sync_jobs\x06\x166" "erp_core_data_sync_jobs\x06\x169" "1"
```

Note, in redis a list with no elements will be automatically removed, so if
consumer is fast enough, you cannot find the queue key in redis.

Also, there is some binding data. Binding data has redis key
`_kombu.binding.<queue>` , the value is `routing key, pattern, queue`. See
[code](https://github.com/celery/kombu/blob/7516daf7a774dd9671d16c475c6ac266de977a0a/kombu/transport/virtual/base.py#L588)
For example,

```
127.0.0.1:6379> SMEMBERS _kombu.binding.celery
1) "celery\x06\x16\x06\x16celery"
```

Here, `\x06\x16\` is the delimiter. So splitting above value, you get
`celery, '', celery`.

One interesting thing about kombu is that it registers redis client socket to
an epoll loop, so calling `BRPOP` is async.

In the above example, the `body` content is base64 encoded. See
[code](https://github.com/celery/kombu/blob/7516daf7a774dd9671d16c475c6ac266de977a0a/kombu/transport/virtual/base.py#L597-L619).

Redis transport support simulating Acknowledge of AMQP protocol. Redis channel
defined another queue `unacked`. When a consumer read a message from a queue,
the message will be move from the queue to `unacked`, and when you call
`message.ack()`, it will be deleted from `unacked`. Redis transport also has a
mechanism to restore unacknowledged messages. It defines a redis zset
`unacked_index`, which stores the delivery_tag of each unacknowledged message
and its timestamp. When the timestamp becomes older enough, the message will be
moved back to its original queue from `unacked`. The default threshold is 1h.
Redis transport will
[repeatably perform this check](https://github.com/celery/kombu/blob/f182a98e75e448b21fb2c12e69fb9565916ea32d/kombu/transport/redis.py#L1317).

Kombu consumer uses callbacks to process messages. There are two ways to
register callbacks. One is `def register_callback`, the other is setting
`on_message` field. If you provided the `on_message` function, then `callbacks`
will not be used. Callbacks take two arguments `(body, message)`, while
`on_message` takes on argument `(message)`, which leaves decoding to user and
is probably more efficient. See more details
[here](https://github.com/celery/kombu/blob/7516daf7a774dd9671d16c475c6ac266de977a0a/kombu/messaging.py#L334).
Celery uses `on_message` instead of callbacks. See
[code](https://github.com/celery/celery/blob/e726978a39a05838805d2b026c4f1c962cfb23b7/celery/worker/loops.py#L60-L61).

Kombu message is defined
[here](https://github.com/celery/kombu/blob/7516daf7a774dd9671d16c475c6ac266de977a0a/kombu/message.py#L18).
It is consumed in celery's worker controller. See more detail below.

### redis broker: `send_command` and `parse_response` ?

Redis implements its own protocol on top of TCP, which is just a bunch of
socket api calls underneath. It works in a request-response pattern. On server
side,

```
recv() from client;
while(condition)
{
  send() from server; //acknowledge to client
  recv() from client;
}
```

On client side,

```
while(condition)
{
  send() from client;
  recv() from server;
}
```

Basically, server side does `recv -> send -> recv -> ..` and client side does
`send -> recv -> send -> ...`. It is important that `send` and `recv`
interleave with each other. It would be a disaster if client does two
consecutive `send`s and only one `recv`, which leads to one message lost. This
potential bug was found in `py-redis`
[issue-977](https://github.com/redis/redis-py/issues/977). After this bug is
fixed, the new implementation always has a `recv` following a `send`. See
[code](https://github.com/redis/redis-py/blob/428510b5786c70a26a8797b8eb33b89777132884/redis/client.py#L1203-L1204).

Ok! Why we care about it? Redis broker in celery does not simply dispatch a
`BRPOP` command. Instead, it registers the underlying socket to `epoll` and
does `send` and `recv` in different code path. So it is extremely important to
make sure the sequence is correct. The relevant source code is
[here](https://github.com/celery/kombu/blob/ec533af9c1c6e156a1fe754fddc2095ebdba8554/kombu/transport/redis.py#L942-L976).
The code author puts a lot of effort to manage variable `self._in_poll` such
that it is always a `send -> recv -> send -> recv -> ...` sequence.

At the beginning of each cycle, the redis broker calls `send_command` to redis
server. This part is implemented as a
[tick cabllack](https://github.com/celery/kombu/blob/717ad3ddc98d83a7b9e2cb7054bb92a0cd6c8d53/kombu/asynchronous/hub.py#L309-L310).
Then `Hub` calls
[epoll.poll()](https://github.com/celery/kombu/blob/717ad3ddc98d83a7b9e2cb7054bb92a0cd6c8d53/kombu/asynchronous/hub.py#L316)
to get ready socket. There are a few possible cases:

1. `BRPOP` socket is not ready. `self._in_poll` is true, so it won't call
   `send_command` in the next cycle.
2. `BRPOP` socket is ready, and QoS `can_consume` returns true.
   `parse_response` is called, and message will be right-popped from redis.
   Also, `self._in_poll` will be set to false. Now you know when exactly the
   message will disappear from the queue.
3. `BRPOP` socket is ready, but QoS `can_consume` returns false. Nothing
   happens. `self._in_poll` is still true and it enters the next cycle.

Case #3 above is what QoS means to AMQP protocol. Only when downstream has
resource to consume message, message will be sent to downstream.

### why there are two polls: Hub and redis

No, there is only on poll inside `Hub.py`. The `poll` instance inside redis
broker refers to the same instance inside `Hub.py`. See
[code](https://github.com/celery/kombu/blob/ec533af9c1c6e156a1fe754fddc2095ebdba8554/kombu/transport/redis.py#L541).

## worker

We should distinguish worker controller and worker. Worker controller is
responsible for dispatching tasks to workers. See below example, PID 1 is the
worker controller.

```
root@celery-critical-short-jobs-f879dfc86-5zzft:/app# ps -efww
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  1 Nov11 ?        00:08:39 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:MainProcess] -active- (-A celery_app worker --loglevel=info --without-mingle --queues=critical_short_jobs --concurrency=8 --prefetch-multiplier=1 -Ofair)
root        38     1  0 Nov11 ?        00:00:04 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-1]
root        50     1  0 Nov11 ?        00:00:02 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-2]
root        62     1  0 Nov11 ?        00:00:02 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-3]
root        74     1  0 Nov11 ?        00:00:02 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-4]
root        86     1  0 Nov11 ?        00:00:02 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-5]
root        98     1  5 Nov11 ?        00:32:44 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-6]
root       110     1  0 Nov11 ?        00:04:13 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-7]
root       122     1  0 Nov11 ?        00:00:20 [celeryd: celery@celery-critical-short-jobs-f879dfc86-5zzft:ForkPoolWorker-8]
```

### Initialization

Default pool mode is
[prefork, i.e., multprocess](https://github.com/celery/celery/blob/47118fbf236a8c1bff7136ef47a797e233593d84/celery/bin/worker.py#L199-L200)

How worker is initialized? If you only look at the
[function signature](https://github.com/celery/celery/blob/47118fbf236a8c1bff7136ef47a797e233593d84/celery/bin/worker.py#L302),
you may think all parameters are defined like `click options`. However, there
are more options than that. If you scroll down a few lines, you see

```
app.config_from_cmdline(ctx.args, namespace='worker')
```

This line introduce more parameters like the default Consumer class.

### How does worker work?

To understand how worker works, you need get familiar with the concept of
[Blueprint](https://docs.celeryq.dev/en/stable/userguide/extending.html). There
are two blueprints: worker and consumer, and consumer is also a bootstep of
worker blueprint.

Using redis transport to illustrate the process.

1. Redis transport epolls `BRPOP` command.
2. Worker controller receive the message, transform the message to a request
   object and emits a `task_received` signal.
   [Code](https://github.com/celery/celery/blob/ad994719bafe6747af6cf8251efb0925284a9260/celery/worker/strategy.py#L146:L165).
   Note, if eta/countdown is set, task will not be executed immediately, but
   instead is reserved and executed later. See
   [code](https://github.com/celery/celery/blob/ad994719bafe6747af6cf8251efb0925284a9260/celery/worker/strategy.py#L199).
   So tasks with eta can be quickly drained in the task queue and moved to
   unacked queue. To tell if a queue is busy or not, you should check both its
   queue and unacked queue.
3. The request is sent to a
   [execution pool](https://github.com/celery/celery/blob/f462a437e3371acb867e94b52c2595b6d0a742d8/celery/worker/worker.py#L223).
   More specifically, it is an async pool, which has a function
   [async_apply](https://github.com/celery/celery/blob/54862310a929fa1543b4ae4e89694905015a1216/celery/worker/request.py#L701:L717)
   that execute the request in a forked process. This part is interesting. It
   turns out Celery does not use python multiprocessing library, it uses its
   own fork called [billiard](https://github.com/celery/billiard). It has
   similar interface as Celery itself, namely, you submit a piece of work to a
   forked process by calling
   [apply_async](https://billiard.readthedocs.io/en/latest/library/multiprocessing.html#using-a-pool-of-workers).

### How does eta/countdown work?

From above section, we know that tasks with eta will be reserved and executed
later, so workers are not blocked. This section talks more details about how
eta works.

[Timer and Hub](https://github.com/celery/celery/blob/d32356c0e46eefecd164c55899f532c2fed2df57/celery/worker/components.py#L32-L75)
are two bluesteps of worker blueprint. Hub constructor takes a timer instance.

> Hub - The workers async event loop

The
[create_loop](https://github.com/celery/kombu/blob/7516daf7a774dd9671d16c475c6ac266de977a0a/kombu/asynchronous/hub.py#L310)
function is basically an iterator and each iteration fires events in Timer's
queue which stores eta tasks. Note, this queue is purely in-memory.

Consumer's last bluesteps is
[EvLoop](https://github.com/celery/celery/blob/117cd9ca410e8879f71bd84be27b8e69e462c56a/celery/worker/consumer/consumer.py#L618).
Here `c.loop` is
[asyncloop](https://github.com/celery/celery/blob/117cd9ca410e8879f71bd84be27b8e69e462c56a/celery/worker/consumer/consumer.py#L212).
Basically, this line start the infinite loop that call
[next(Hub.\_loop)](https://github.com/celery/celery/blob/e726978a39a05838805d2b026c4f1c962cfb23b7/celery/worker/loops.py#L97).
That is all the pieces to understand how eta tasks work.

### How does server side rate limit work?

Task can be rate limited on server side. See
[this code](https://github.com/celery/celery/blob/ad994719bafe6747af6cf8251efb0925284a9260/celery/worker/strategy.py#L202).
Celery consumer implements a simple
[token bucket rate limit algorithm](https://github.com/celery/kombu/blob/7516daf7a774dd9671d16c475c6ac266de977a0a/kombu/utils/limits.py#L11).
When consumer receives a task and it does not enough tokens, then the task will
be put the `timer` with an expected wait time. So in this sense, rate-limited
tasks behave similar to eta tasks.

### Reserve task

Each task controller holds two in-memory containers:
[requests and reserved_requests](https://github.com/celery/celery/blob/d29610bac81a1689b53440e6347b9c5ced038751/celery/worker/state.py#L49).
How do they behave during warm shutdown?

`maybe_shutdown` called twice: `loops.py` and `consumer.py`.

### Drop message during shutdown

Using redis as message broker, celery can potentially drop messages during
shutdown. See this [issue](https://github.com/celery/celery/issues/4354).

### solo vs prefork

In `solo` mode, worker controller itself executes the task. In `prefork` mode,
worker controller hands the task to worker processes.

## Task

When we add annotation `@celery_app.task` to a function, it registers this
function to the task list inside `celery_app`. See this
[code](https://github.com/celery/celery/blob/45b5c4a1d4c0c099fc4ccd13fc4c80e2ccedc088/celery/app/base.py#L442).
You can see that the task is not immediately instantiated. It is stored inside
a `PromiseProxy` and will be instantiated once we call the
[finalize function](https://github.com/celery/celery/blob/45b5c4a1d4c0c099fc4ccd13fc4c80e2ccedc088/celery/app/base.py#L514).
As you can see, finalize function instantiates all registered tasks in one
stroke. So when is it called? The answer is it is called the first time you try
to look up a task in the task registry. See
[this part](https://github.com/celery/celery/blob/45b5c4a1d4c0c099fc4ccd13fc4c80e2ccedc088/celery/app/base.py#L1298).

I was blocked by this part for one hour because I was writing a test that
switches Celery Task type. It behaviors differently when I (un)commented one
line that prints out the details of a specific task. OK, so the I believe, my
"observation" must changed something and I need to read the source code!

## resources

1. useful management commands
   https://docs.celeryq.dev/en/latest/userguide/monitoring.html#management-command-line-utilities-inspect-control

## signals

Use signals to define hooks that pre/post run of a job, or even configure
logging format by `set_logging` signal.

## Canvas

### chain

Chain is recursively called at
[this place](https://github.com/celery/celery/blob/59263b0409e3f02dc16ca8a3bd1e42b5a3eba36d/celery/app/trace.py#L507).

> 1704750305.685808 [0 127.0.0.1:52368] "LPUSH" "celery" "{\"body\":
> \"W1szMF0sIHt9LCB7ImNhbGxiYWNrcyI6IG51bGwsICJlcnJiYWNrcyI6IG51bGwsICJjaGFpbiI6IFt7InRhc2siOiAidGFza3Muc2xhY2tfdGFza3MuZG9fc2xlZXAiLCAiYXJncyI6IFs2MF0sICJrd2FyZ3MiOiB7fSwgIm9wdGlvbnMiOiB7InRhc2tfaWQiOiAiOTc4OWM3ZTItNzI3NC00ZmEyLWEwZWUtMzU1Y2ZkMjAxMDI3IiwgInJlcGx5X3RvIjogImQxYjk0YTRlLWMyYmUtMzdmZS04NWVjLWY3MDY4YzRlNTI0MCJ9LCAic3VidGFza190eXBlIjogbnVsbCwgImltbXV0YWJsZSI6IGZhbHNlfV0sICJjaG9yZCI6IG51bGx9XQ==\",
> \"content-encoding\": \"utf-8\", \"content-type\": \"application/json\",
> \"headers\": {\"lang\": \"py\", \"task\": \"tasks.slack_tasks.do_sleep\",
> \"id\": \"b8fb8776-9746-45bd-80a6-dd29856c41ad\", \"shadow\": null, \"eta\":
> null, \"expires\": null, \"group\": null, \"group_index\": null, \"retries\":
> 0, \"timelimit\": [null, null], \"root_id\":
> \"b8fb8776-9746-45bd-80a6-dd29856c41ad\", \"parent_id\": null, \"argsrepr\":
> \"(30,)\", \"kwargsrepr\": \"{}\", \"origin\": \"gen5918@ip-172-30-7-82\",
> \"ignore_result\": false, \"replaced_task_nesting\": 0, \"stamped_headers\":
> null, \"stamps\": {}, \"x_request_id\": null}, \"properties\":
> {\"correlation_id\": \"b8fb8776-9746-45bd-80a6-dd29856c41ad\", \"reply_to\":
> \"d1b94a4e-c2be-37fe-85ec-f7068c4e5240\", \"\_flask_request_context\": {},
> \"pre_enqueue_timestamp\": {\"**type**\": \"datetime\", \"**value**\":
> \"2024-01-08T21:41:08.479523\"}, \"delivery_mode\": 2, \"delivery_info\":
> {\"exchange\": \"\", \"routing_key\": \"celery\"}, \"priority\": 0,
> \"body_encoding\": \"base64\", \"delivery_tag\":
> \"e4fb5546-d496-4d70-a233-6f1a81cb7ff9\"}}"

The body is actually

```bash
$ echo -n W1szMF0sIHt9LCB7ImNhbGxiYWNrcyI6IG51bGwsICJlcnJiYWNrcyI6IG51bGwsICJjaGFpbiI6IFt7InRhc2siOiAidGFza3Muc2xhY2tfdGFza3MuZG9fc2xlZXAiLCAiYXJncyI6IFs2MF0sICJrd2FyZ3MiOiB7fSwgIm9wdGlvbnMiOiB7InRhc2tfaWQiOiAiOTc4OWM3ZTItNzI3NC00ZmEyLWEwZWUtMzU1Y2ZkMjAxMDI3IiwgInJlcGx5X3RvIjogImQxYjk0YTRlLWMyYmUtMzdmZS04NWVjLWY3MDY4YzRlNTI0MCJ9LCAic3VidGFza190eXBlIjogbnVsbCwgImltbXV0YWJsZSI6IGZhbHNlfV0sICJjaG9yZCI6IG51bGx9XQ== | base64 -d

[[30], {}, {"callbacks": null, "errbacks": null, "chain": [{"task": "tasks.slack_tasks.do_sleep", "args": [60], "kwargs": {}, "options": {"task_id": "9789c7e2-7274-4fa2-a0ee-355cfd201027", "reply_to": "d1b94a4e-c2be-37fe-85ec-f7068c4e5240"}, "subtask_type": null, "immutable": false}], "chord": null}]2024-01-08 13:47:19 (bash) ~
$ cd code/kombu/
```

I was wondering how it (de)serializes the messages. I found that signature
defines a
[json](https://github.com/celery/celery/blob/ec3714edf37e773ca5372f71f7f4ee5b1b33dd5d/celery/canvas.py#L618)  
method, but I did not find any PEP introduces this magic method. Then I find
this
[line of code](https://github.com/celery/celery/blob/ec3714edf37e773ca5372f71f7f4ee5b1b33dd5d/celery/canvas.py#L618).
Hmm, it is Celery specific thing. I feel it is bad to name it this way. It will
confuse users.

#### mutable vs immutable

In a chain, you can choose if the child task uses the result of parent task's
return value as the first argument or not. This is accomplished by `task.s()`
and `task.si()` respectively. See
[code](https://github.com/celery/celery/blob/ec3714edf37e773ca5372f71f7f4ee5b1b33dd5d/celery/canvas.py#L829).

#### exception propagation

If a parent task failed, the child tasks won't be scheduled, but the same
failure message will be stored in backend store as well. See
[code](https://github.com/celery/celery/blob/1c4ff33bd22cf94e297bd6449a06b5a30c2c1fbc/celery/backends/base.py#L182).
