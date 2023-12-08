---
layout: post
title: Spinnaker igor
date: 2023-12-08 10:00 -0800
categories: [spinnaker, component]
tags: [spinnaker]
---

I thought `igor` may stands for "integration ..." in spinnaker. Then I asked
chatgpt.

> In Spinnaker, "Igor" does not stand for an acronym. Igor is the name given to
> the component responsible for managing the execution of pipelines in
> Spinnaker. It is named after the hunchbacked assistant of Dr. Frankenstein in
> Mary Shelley's novel "Frankenstein." The name Igor is often associated with a
> loyal and diligent helper, reflecting the role of the component in managing
> deployments and pipeline execution in Spinnaker.

Ok. TIL!

Igor is a service that provides a single point of integration with Continuous
Integration (CI) and Source Control Management (SCM) services for Spinnaker.

## How to build

Igor does not build with openjdk-19.

```
./gradlew build -x test  -Dorg.gradle.java.home=/Users/xiongding/Library/Java/JavaVirtualMachines/azul-15.0.10/Contents/Home
```

Choose accordingly version in Intellij too.

## Config file

```
PID   USER     TIME  COMMAND
    1 spinnake  1:27 java -Djava.security.egd=file:/dev/./urandom -Dspring.config.additional-location=/opt/spinnaker/config/
```

The config file is at `/opt/spinnaker/config/igor.yml`. Parts of this config
files are below

```
...
jenkins:
  enabled: true
  masters:
  - name: my-jenkins-master
    permissions: {}
    address: https://jenkins.tryevergreen.com/
    username: admin
    password: <...>
  - name: private-jenkins-master
    permissions: {}
    address: http://jenkins.devops:8080
    username: admin
    password: <...>
...
```

## Jenkins

Spinnaker can start a Jenkins a build. The interface is
[here](https://github.com/spinnaker/igor/blob/2f6f3951d1e0710e63bdf1dffdd8bef72345d102/igor-web/src/main/groovy/com/netflix/spinnaker/igor/jenkins/client/JenkinsClient.groovy#L76-L76).
