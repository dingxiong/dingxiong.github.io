---
layout: post
title: Java -- Getting Started
date: 2024-03-01 00:30 -0800
categories: [programming-language, java]
tags: [java]
---

## Setting up Java in MacOS

MacOS is notorious for the lack of transparency of how everything works. Java
is no exception. With a brand new MacBook, you have a few pre-installed
commands: `/usr/bin/java`, `/usr/bin/javac`, `/usr/libexec/java_home` and etc.
Do not try to figure out the source code of these commands. They are built-in
scripts written in Objective-C or Swift using Apple's Foundation framework. Its
source code is not publicly available. You can use `otool -L` to list the
shared libs used by them. Most likely, you can find
`/System/Library/PrivateFrameworks` in the output.

The first step is installing a jdk. Let's do it with `brew install openjdk`.
The output has a caveat section.

```
==> Caveats
For the system Java wrappers to find this JDK, symlink it with
  sudo ln -sfn /opt/homebrew/opt/openjdk/libexec/openjdk.jdk /Library/Java/JavaVirtualMachines/openjdk.jdk
```

To understand this caveat, we need to understand how MacOS finds the correct
version of Java. `/usr/bin/java` is a shim command, and from my
observation/guess, it chooses the java version in two steps:

1. If `JAVA_HOME` is set, then it delegates to `$JAVA_HOME/bin/java`.
2. If `JAVA_HOME` is absent, then it calls `/usr/libexec/java_home` to find the
   most appropriate java home.

How does `/usr/libexec/java_home` find the available java homes? Again, from
the information I collected online and my guess, it searches a few root
folders: `/Library/Java/JavaVirtualMachines/`,
`/System/Library/Java/JavaVirtualMachines/` and
`~/Library/Java/JavaVirtualMachines/`. Every sub folder of them potentially
contains a valid java jdk folder. `/usr/libexec/java_home` looks for a file
called `Contents/Info.plist` which contains meta data about this jdk version.
Then it sorts them by version reversely and the highest version would be used
for `JAVA_HOME` if `JAVA_HOME` is not explicitly specified. For example,

```
/usr/libexec/java_home -V
Matching Java Virtual Machines (6):
    23.0.2 (arm64) "Homebrew" - "OpenJDK 23.0.2" /opt/homebrew/Cellar/openjdk/23.0.2/libexec/openjdk.jdk/Contents/Home
    19 (arm64) "Oracle Corporation" - "OpenJDK 19" /Users/xiongding/Library/Java/JavaVirtualMachines/openjdk-19/Contents/Home
    18.0.2.1 (arm64) "Eclipse Adoptium" - "OpenJDK 18.0.2.1" /Users/xiongding/Library/Java/JavaVirtualMachines/temurin-18.0.2.1/Contents/Home
    17.0.4 (arm64) "Amazon.com Inc." - "Amazon Corretto 17" /Users/xiongding/Library/Java/JavaVirtualMachines/corretto-17.0.4.1/Contents/Home
    15.0.10 (arm64) "Azul Systems, Inc." - "Zulu 15.46.17" /Users/xiongding/Library/Java/JavaVirtualMachines/azul-15.0.10/Contents/Home
    13.0.14 (arm64) "Azul Systems, Inc." - "Zulu 13.54.17" /Users/xiongding/Library/Java/JavaVirtualMachines/azul-13.0.14/Contents/Home
/opt/homebrew/Cellar/openjdk/23.0.2/libexec/openjdk.jdk/Contents/Home
```

It shows I have a few versions of jdk installed and `openjdk/23.0.2` will be
used for `/usr/bin/java` if `JAVA_HOME` is not specified. See result below.

```
$ JAVA_HOME='' /usr/bin/java --version
openjdk 23.0.2 2025-01-21
OpenJDK Runtime Environment Homebrew (build 23.0.2)
OpenJDK 64-Bit Server VM Homebrew (build 23.0.2, mixed mode, sharing)

$ JAVA_HOME='/Users/xiongding/Library/Java/JavaVirtualMachines/azul-13.0.14/Contents/Home' /usr/bin/java --version
openjdk 13.0.14 2023-01-17
OpenJDK Runtime Environment Zulu13.54+17-CA (build 13.0.14+5-MTS)
OpenJDK 64-Bit Server VM Zulu13.54+17-CA (build 13.0.14+5-MTS, mixed mode)
```

Let's come back to the `brew install` message above. `Homebrew` is
conservative. It installs jdk inside folder `/opt/homebrew/opt`, so it is
user's responsibility to soft link it to the
`/Library/Java/JavaVirtualMachines/`. I think most likely we would put it under
the home library folder not the root system folder.

```
ln -sfn /opt/homebrew/opt/openjdk/libexec/openjdk.jdk ~/Library/Java/JavaVirtualMachines/openjdk.jdk
```

To sum up, Setting up Java in MacOS needs three steps

```
brew install openjdk
ln -sfn /opt/homebrew/opt/openjdk/libexec/openjdk.jdk ~/Library/Java/JavaVirtualMachines/openjdk.jdk
echo 'export JAVA_HOME=/opt/homebrew/opt/openjdk/libexec/openjdk.jdk/Contents/Home/' >> ~/.bashrc
```

## Reading Openjdk

[openjdk](https://github.com/openjdk/jdk) is the starting point. However,
IntelliJ fails to resolve symbols when open it directly. Actually, openjdk team
provides some guidance on how to set up the repo in IntelliJ. See
[doc](https://github.com/openjdk/jdk/blob/master/doc/ide.md#intellij-idea).

```bash
bash configure
bash bin/idea.sh
```

I encountered a few errors when running above two commands in a Macbook M1. In
the configure step,

```
configure: error: XCode tool 'metal' neither found in path nor with xcrun
```

This [post](https://github.com/gfx-rs/gfx/issues/2309) helps. Basically, we
need to change the active developer directory of xcode. We can change it back
once the configuration stage is done.

The second step requires ant: `brew install ant`.

However, after all these steps, it still does not work. Finally, I use a simple
trick: `File -> Project Structure -> Modules -> Add an existing JDK`.

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
