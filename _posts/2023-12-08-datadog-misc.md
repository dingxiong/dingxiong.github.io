---
layout: post
title: Datadog misc
date: 2023-12-08 09:56 -0800
categories: [datadog misc]
tags: [datadog]
---

## Features of datadog

- Unified service tagging

## Integration

Autodiscovery integration It provides a lot of integrations for popular tools,
applications to collect their metrics and logs. See
https://docs.datadoghq.com/agent/kubernetes/integrations/?tab=kubernetes and
https://github.com/DataDog/integrations-core. However, I followed with the doc
above to set up a customized metric integration. It did not work. The error
message is

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

## APM

APM concepts:

- Span name or Operation name in the datadog traces explorer.
- Resource name: If not provided, it will be the same as span name. See
  [code](https://github.com/DataDog/dd-trace-py/blob/fdb82b092714a75192a09957ac151e849955c77f/ddtrace/span.py#L138)

Datadog APM has a few components. Below is the summary of each component does.

### Continuous profiling

DD continuous profiler is a module to collect system information like CPU
usage, memory footprint, error stack trace, and etc. See
[this introduction](https://www.datadoghq.com/blog/engineering/how-we-wrote-a-python-profiler/).
In
[ddtrace](https://github.com/DataDog/dd-trace-py/blob/9b05c3b873eafb1e4a98824a06a4f03b98b27973/ddtrace/profiling/collector/stack.pyx#L476)
lib, this part is under `profiling` folder.

Also, note that, for performance consideration, most collectors are written in
Cython.

## Debug tips

- Each patched application will have an attribute `__datadog_patch`
- change logLevel to DEBUG in the helm yaml file.

## Tools

- datadog agent commands: Ex:
  `docker exec -it <AGENT_CONTAINER_NAME> agent configcheck -v`
- APM related dashboards. Go to datadog dashboard page and search for `APM`,
  you will see a few ingestion related dashboards.
