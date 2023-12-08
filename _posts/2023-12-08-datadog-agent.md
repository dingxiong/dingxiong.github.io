---
title: Datadog agent
date: 2023-12-08 09:56 -0800
categories: [datadog]
tags: [datadog]
---

Datadog agent has many components. First, a datadog agent should run in the
host. Then a datadog client send data to this agent in the code.

- [datadog](https://github.com/DataDog/datadogpy): datadog agent client. It
  deals with connection options
- [ddtrace](https://github.com/DataDog/dd-trace-py): datadog APM. Send traffic
  specific to logging, traces, services.

## Datadog setup

You can check the datadog configuration in the pod directly
`preintenv | grep DD_`. These variables are injected to datadog agent during
startup. For example, below code will use env variable
`DD_CONTAINER_INCLUDE_LOGS` for this option.

```
config.BindEnvAndSetDefault("container_include_logs", []string{})
```

It has a step to add a `envPrefix` `DD_` to all options.

A few main configurations below.

- `DD_ENV`
- `DD_SERVICE` defines the service name in the APM services page.
- `DD_VERSION`

## Agent

Below are the processes running in a datadog container.

```
root@datadog-27q4r:/# ps -efww
UID        PID  PPID  C STIME TTY          TIME CMD
root         1     0  3 Jan09 ?        00:15:04 agent run
root      3371     0  0 02:31 pts/0    00:00:00 bash
root      3485  3371  0 02:31 pts/0    00:00:00 ps -efww
```

The command-line program `agent run` is a cobra program and `run` is a
subcommand!. The entry point is
[here](https://github.com/DataDog/datadog-agent/blob/36e4383d4f8461e0e2342533dddac0eab0ec25db/cmd/agent/app/run.go#L93).
As you can imagine, this `run` subcommand does a lot of things. Let's dive into
each sub component one by one.

### Log agent

Datadog is installed as a DaemonSet in Kubernetes and if enabled, it
automatically tails container logs and sends them to datadog.com. So how does
this process work?

The log-agent is started
[here](https://github.com/DataDog/datadog-agent/blob/36e4383d4f8461e0e2342533dddac0eab0ec25db/cmd/agent/app/run.go#L426-L435).
It is controlled by a config `logs_enabled`.

Log-agent consists of a set of launchers. Each launcher is responsible for a
certain category of logs. We are interested in docker
[file logs](https://github.com/DataDog/datadog-agent/blob/36e4383d4f8461e0e2342533dddac0eab0ec25db/pkg/logs/agent.go#L72).
From there, you can see how file trailers are set up. The basic flow is as
follows,

```
Trailer calls decoder to decode file content line by line.
  -> decoder calls parser to pare line.
```

The format for docker logs is
[below](https://github.com/DataDog/datadog-agent/blob/166e5353db6beb417cd8b481af253576094d05ae/pkg/logs/internal/parsers/dockerfile/docker_file.go#L36).

```
type logLine struct {
	Log    string
	Stream string
	Time   string
}
```

Here, `stream` is either `stderr` or `stdout`, and datadog uses this field to
determine if the log level is an error or info, so you application's log stream
channel matters!
