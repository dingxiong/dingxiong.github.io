---
layout: post
title: Datadog misc
date: 2023-12-08 09:56 -0800
categories: [datadog]
tags: [datadog]
---

## Features of datadog

- Unified service tagging

## Integration

Autodiscovery integration It provides a lot of integrations for popular tools,
applications to collect their metrics and logs. See
[doc](https://docs.datadoghq.com/agent/kubernetes/integrations/?tab=kubernetes)
and [code](https://github.com/DataDog/integrations-core). However, I followed
with the doc above to set up a customized metric integration. It did not work.
The error message is

```
2022-05-15 02:47:28 UTC | CORE | DEBUG | (pkg/collector/python/loader.go:148 in Load) | Unable to load python module - datadog_checks.python-test-123: unable to import module 'datadog_checks.python-test-123': No module named 'datadog_checks.python-test-123'
2022-05-15 02:47:28 UTC | CORE | DEBUG | (pkg/collector/python/loader.go:148 in Load) | Unable to load python module - python-test-123: unable to import module 'python-test-123': No module named 'python-test-123'
2022-05-15 02:47:28 UTC | CORE | DEBUG | (pkg/collector/python/loader.go:156 in Load) | PyLoader returning unable to import module 'python-test-123': No module named 'python-test-123' for python-test-123
2022-05-15 02:47:28 UTC | CORE | ERROR | (pkg/collector/scheduler.go:201 in getChecks) | Unable to load a check from instance of config 'python-test-123': JMX Check Loader: check is not a jmx check, or unable to determine if it's so; Python Check Loader: unable to import module 'python-test-123': No module named 'python-test-123'; Core Check Loader: Check python-test-123 not found in Catalog
2022-05-15 02:47:28 UTC | CORE | ERROR | (pkg/collector/scheduler.go:248 in GetChecksFromConfigs) | Unable to load the check: unable to load any check from config 'python-test-123'
```

So it is more complicated to use this feature.

### How integration works

Datadog is funny. The integration logic is written in Python, but the execution
is written in C.

Use Nginx as an example, go to `integrations-core/nginx`
[folder](https://github.com/DataDog/integrations-core/blob/e0420b8f9afe7b1c0e6dfd374b1f0c458a1f8eb9/nginx/datadog_checks/nginx/nginx.py#L29).
You will see `class Nginx(AgentCheck):`, which has a `def run(self)`
[method](https://github.com/DataDog/integrations-core/blob/7d15bd83bad8762f1b64acc46e6fa39b81e66c30/datadog_checks_base/datadog_checks/base/checks/base.py#L1085)
to generates all Nginx metrics. Then where this `run` mothod is called? I
cannot find it anywhere in this repo. Wired, right? Actually, it turns out that
it is called in C program in
[datadog-agent repo](https://github.com/DataDog/datadog-agent/blob/336d77ebceec4734d0349b985ffa4f32456954ac/rtloader/three/three.cpp#L386).

All integrations have a corresponding `AgentCheck` class. From there, you can
easily see what metrics are defined.

OK. Then comes to the next question: where do these integrations run? I am
using datadog installed in a K8S cluster using helm chart as an example. Some
key configurations of this helm chart is as follows,

```
datadog:
  clusterChecks:
    enabled: true
clusterAgent:
  enabled: true
  replicas: 2
clusterChecksRunner:
  enabled: true
  replicas: 2
```

So we will have 2 cluster agents, 2 cluster checks runners and a few datadog
agent forming a daemonset.

```
datadog-cluster-agent-665f97bdc7-8r84k        1/1     Running   0          9h
datadog-cluster-agent-665f97bdc7-mxl6h        1/1     Running   0          9h
datadog-clusterchecks-6b74dd667b-n4j78        1/1     Running   0          9h
datadog-clusterchecks-6b74dd667b-rxjww        1/1     Running   0          9h
datadog-cts4p                                 3/3     Running   0          3h3m
datadog-d9jxn                                 3/3     Running   0          9h
...
```

The daemonset pods are not interesting. There is an agent pod running in each
node to collect node/pod specific metrics. Cluster agent and cluster checks
runner are more interesting.

[datadog-cluster-agent](https://github.com/DataDog/datadog-agent/blob/083a2213e83d8e845feceaae17f50b6753a95f98/cmd/cluster-agent/app/app.go#L60)
runs in the cluster agent pod. Note, even we have two cluster agents in this
setup. Only one is leader, the other is a stand-by. You can see which one is
the leader by the `status` subcommand.

```
root@datadog-cluster-agent-665f97bdc7-8r84k:/# datadog-cluster-agent status
...
Leader Election
===============
  Leader Election Status:  Running
  Leader Name is: datadog-cluster-agent-665f97bdc7-8r84k
  Last Acquisition of the lease: Wed, 10 Jul 2024 17:55:32 UTC
  Renewed leadership: Thu, 11 Jul 2024 03:17:05 UTC
  Number of leader transitions: 220 transitions
...
```

Cluster checks runners are kind of normal runners and just run `agent run`
command inside it. However, they are used to monitor external resources and not
tied to application nodes/pods, so they do not form a daemonset. Cluster agent
figures out how many `AgentCheck` jobs and dispatch these jobs to cluster
checks runners. For example, in this example setting, support we have three
check jobs: Mysql metrics, Postgress metrics, and Elasticsearch metrics, then
cluster agent may dispatch Mysql and Postgres check jobs to cluster checks
runner #1 and Elasticsearch check job to cluster checks runner #2. We can check
the task distribution as follows:

```
root@datadog-cluster-agent-665f97bdc7-8r84k:/# datadog-cluster-agent clusterchecks
=== 2 agents reporting ===

Name                                     Running checks
datadog-clusterchecks-6b74dd667b-n4j78   1
datadog-clusterchecks-6b74dd667b-rxjww   2

===== Checks on datadog-clusterchecks-6b74dd667b-n4j78 =====

=== elastic check ===
...
===

===== Checks on datadog-clusterchecks-6b74dd667b-rxjww =====

=== mysql check ===
...
===

=== postgres check ===
...
===
...
```

Here is a
[diagram](https://github.com/DataDog/datadog-agent/blob/083a2213e83d8e845feceaae17f50b6753a95f98/pkg/clusteragent/clusterchecks/README.md#L30)
illustrates the relationship.

### Postgres Integration

Datadog Postgres integration has the ability to run explanation on sample
queries.

First, what are the sample queries? These are queries captured in side
`pg_stat_activity`.

Second, how does it run the `explain` statement? Datadog requires you setting
up a function `datadog.explain_statement` to run the `explain` statement. See
[instruction](https://docs.datadoghq.com/database_monitoring/setup_postgres/aurora/?tab=kubernetes)
and
[code](https://github.com/DataDog/integrations-core/blob/6ba6a69f98c64679ed06e0e92c8e96fbaadc3287/postgres/datadog_checks/postgres/statement_samples.py#L173).
Note, we must create this function in every database we want to capture the
queries. Datadog has a step to test whether this function exist or not. If not,
it will exit early. See
[code](https://github.com/DataDog/integrations-core/blob/6ba6a69f98c64679ed06e0e92c8e96fbaadc3287/postgres/datadog_checks/postgres/statement_samples.py#L605).
Below are some error logs captured because of this mistake.

```
2024-07-11 01:43:01 UTC | CORE | WARN | (pkg/collector/python/datadog_agent.go:124 in LogMessage) | postgres:14bfc3bc7d98044a | (statement_samples.py:449) | cannot collect execution pl
ans due to invalid schema in dbname=admincoin: InvalidSchemaName('schema "datadog" does not exist\nLINE 1: SELECT datadog.explain_statement($stmt$SELECT * FROM pg_stat...\n               ^\n')
```

### Kafka Integration

Kafka has two types of integration: broker metric integration and consumer
metric integration.

The former requires the Kafka cluster has JMX enabled. The second uses an admin
client to consumer offset, lag metric etc. See code
[here](https://github.com/DataDog/integrations-core/blob/6ba6a69f98c64679ed06e0e92c8e96fbaadc3287/kafka_consumer/datadog_checks/kafka_consumer/kafka_consumer.py#L15).

### kubernetes Integration

How to get cluster name?

When cluster agent starts, it tries to auto detect
[cluster name](https://github.com/DataDog/datadog-agent/blob/083a2213e83d8e845feceaae17f50b6753a95f98/cmd/cluster-agent/app/app.go#L266).
For AWS, what it actually does it calling the instance metadata endpoint

```
curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"

curl http://169.254.169.254/latest/dynamic/instance-identity/document -H "X-aws-ec2-metadata-token: AQAEAEH8bTZXavSdO5XMdOi0iJXNML14UiVC1ZkHwbOppKdUZr9vXQ=="
{
  "accountId" : "242230929264",
  "architecture" : "x86_64",
  "availabilityZone" : "us-east-2b",
  "billingProducts" : null,
  "devpayProductCodes" : null,
  "marketplaceProductCodes" : null,
  "imageId" : "ami-0f98fd42429c01a3c",
  "instanceId" : "i-03cc7ea13c98d1906",
  "instanceType" : "t3.xlarge",
  "kernelId" : null,
  "pendingTime" : "2024-07-16T21:13:33Z",
  "privateIp" : "172.31.79.158",
  "ramdiskId" : null,
  "region" : "us-east-2",
  "version" : "2017-09-30"
}

aws ec2 describe-tags --filters Name=resource-id,Values=i-03cc7ea13c98d1906 --profile=admin
{
    "Tags": [
        {
            "Key": "aws:autoscaling:groupName",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "eks-ng-system-56c62341-1b39-1572-903a-f061ad1ed353"
        },
        {
            "Key": "aws:ec2launchtemplate:id",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "lt-0ddcba2c6b2b08aa2"
        },
        {
            "Key": "aws:ec2launchtemplate:version",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "3"
        },
        {
            "Key": "aws:eks:cluster-name",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "evergreen"
        },
        {
            "Key": "eks:cluster-name",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "evergreen"
        },
        {
            "Key": "eks:nodegroup-name",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "ng-system"
        },
        {
            "Key": "k8s.io/cluster-autoscaler/enabled",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "true"
        },
        {
            "Key": "k8s.io/cluster-autoscaler/evergreen",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "owned"
        },
        {
            "Key": "kubernetes.io/cluster/evergreen",
            "ResourceId": "i-03cc7ea13c98d1906",
            "ResourceType": "instance",
            "Value": "owned"
        }
    ]
}
```

IP 169.254.169.254 is a special
[internal IP](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instancedata-data-retrieval.html#instance-metadata-returns).
I talked about this IP in the k8s notes as well. So basically, you can get the
cluster name and other info from inside cluster agent. However, the current
implementation does not work because it lacks the step to obtain the token,
then the instance metadata call fails with a 401 error.

## APM

APM concepts:

- Span name or Operation name in the datadog traces explorer.
- Resource name: If not provided, it will be the same as span name. See
  [code](https://github.com/DataDog/dd-trace-py/blob/fdb82b092714a75192a09957ac151e849955c77f/ddtrace/span.py#L138)

Datadog APM has a few components. Below is the summary of each component does.

How does APM send the data to datadog?

There are two ways to send data from a localhost to datadog. One is sending
requests to datadog server directly. The other approach is through dd agent.
Initially, I thought both ways should work, so can save the effort of
installing datadog agent. After reading some `dd-trace` code, I find dd agent
is the only option. in `dd-trace`, class
[TraceWriter](https://github.com/DataDog/dd-trace-py/blob/08cc2b61a11d493e801026ab1e82e678d92cfaad/ddtrace/internal/writer/writer.py#L106)
is responsible for sending data to datadog. If you read the hierarchy of this
class, you realize that `AgentWriter` is the only option.

### Continuous profiling

DD continuous profiler is a module to collect system information like CPU
usage, memory footprint, error stack trace, and etc. See
[this introduction](https://www.datadoghq.com/blog/engineering/how-we-wrote-a-python-profiler/).
In
[ddtrace](https://github.com/DataDog/dd-trace-py/blob/9b05c3b873eafb1e4a98824a06a4f03b98b27973/ddtrace/profiling/collector/stack.pyx#L476)
lib, this part is under `profiling` folder.

Also, note that, for performance consideration, most collectors are written in
Cython.

One note about how memory profiler is implemented in dd-trace-py. This is the
[main file](https://github.com/DataDog/dd-trace-py/blob/08cc2b61a11d493e801026ab1e82e678d92cfaad/ddtrace/profiling/collector/_memalloc.c#L432).
Basically, it provides its own version of malloc and free. This is a good
example for us to learn how to write a Python extension in pure C.

## Debug tips

- Each patched application will have an attribute `__datadog_patch`
- change logLevel to DEBUG in the helm yaml file.

## Tools

- datadog agent commands: Ex:
  `docker exec -it <AGENT_CONTAINER_NAME> agent configcheck -v`
- APM related dashboards. Go to datadog dashboard page and search for `APM`,
  you will see a few ingestion related dashboards.
