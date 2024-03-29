---
layout: post
title: Java random stuff
date: 2024-03-01 00:30 -0800
categories: [programming-language, java]
tags: [java]
---

## Debug

I am testing some Kafka Connect plugin recently. Instead of using `print` to
debug, I really wish Java can provide some functionality similar to a dynamic
language such as I insert a `breakpoint()` in the code, then the running
process will jump to a REPL shell.

Then I realize Java provides remote debugging, or more specifically, JDWP (Java
Debug Wire Protocol). Basically, when we start the Java program, we add some
command line arguments to open some socket to allow remote debugger to attach.
For the case of Kafka Connect, we can enable it by setting the `KAFKA_DEBUG`
environment variable. See
[code](https://github.com/apache/kafka/blob/2.8.1/bin/kafka-run-class.sh#L245-L245).

```
-agentlib:jdwp=transport=dt_socket,server=y,address=5005
```

`server=y` means that it starts a server and waits for client to connect.
`server=n` has reverse meaning.

Then we can use jdb to connect to this process

```
$ jdb -attach 127.0.0.1:5005

Initializing jdb ...

> help
** command list **
connectors                -- list available connectors and transports in this VM

run [class [args]]        -- start execution of application's main class

threads [threadgroup]     -- list threads in threadgroup. Use current threadgroup if none specified.
thread <thread id>        -- set default thread
suspend [thread id(s)]    -- suspend threads (default: all)
resume [thread id(s)]     -- resume threads (default: all)
where [<thread id> | all] -- dump a thread's stack
wherei [<thread id> | all]-- dump a thread's stack, with pc info
up [n frames]             -- move up a thread's stack
down [n frames]           -- move down a thread's stack
kill <thread id> <expr>   -- kill a thread with the given exception object
interrupt <thread id>     -- interrupt a thread
...
```

It is very similar to gdb. You can set breakpoint, print out thread stack
trace. And I think it has better support for debugging multi-thread program.

A few notes about breakpoints. First, we must use full class path such as
`stop at org.opensearch.cluster.ClusterState:706`. Second, for inner class, we
should do `stop at <full_class_path>$<inner_class_name>:line_number`. For
example, `stop at org.opensearch.cluster.ClusterState$Builder:706`. You can
inspect the outer class by `class <full_class_path>`.

## Cross compilation

`javac` has three options related to cross compilation: `-source`, `-target`
and `-releaes`. [JEP 247](https://openjdk.org/jeps/247) has concise description
about their usage.
