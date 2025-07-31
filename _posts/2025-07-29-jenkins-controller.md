---
layout: post
title: Jenkins -- Controller
date: 2025-07-29 16:41 -0700
categories: [devops, cicd]
tags: [devops, jenkins]
---

Jenkins is one of the most sophisticated Java code bases I have ever read so
far. I feel it is harder to read even than the Spring codebase. It is a
combination of DSL parser, annotation processing, remote JVM, sandboxing, web
design and etc. You can learn a lot of skills from this project. Also, the huge
amount of must-have plugins are also intimidating for beginners. I hope this
post can shed some light on the Jenkins internals.

By the way, I found its official
[plugin tutorial](https://www.jenkins.io/doc/developer/tutorial/) is super
helpful to grasp some basic idea about `Action` and `jelly`. Try it out if you
haven't!

First, we should know how Jenkins runs and what components it has. Jenkins has
two roles: controller and worker. Controller is responsible for managing all
definitions and states. Worker is responsible for executing jobs. Most
complicated things happen inside the controller. The root entry for the
controller code is
[Jenkins.java](https://github.com/jenkinsci/jenkins/blob/235d237f32835ccc1064c95c701f331d7dece7a3/core/src/main/java/jenkins/model/Jenkins.java).
The controller maintains a list of
[jobs](https://github.com/jenkinsci/jenkins/blob/235d237f32835ccc1064c95c701f331d7dece7a3/core/src/main/java/hudson/model/Job.java).
A job can be executed. Each single execution is called a
[Run/Build](https://github.com/jenkinsci/jenkins/blob/235d237f32835ccc1064c95c701f331d7dece7a3/core/src/main/java/hudson/model/Run.java).
All these entities are dumped to disk as
`${JENKINS_HOME}/jobs/<job-name>/builds/<build-id>`. For example, this
[build.xml](/assets/raw/jenkins_build.xml.txt) is a sample run of a Jenkins
pipeline.

| Annotation  | Purpose                        | Resulting Endpoint                           |
| ----------- | ------------------------------ | -------------------------------------------- |
| `@Exported` | Included in `/api/json` output | `/.../api/json?tree=fieldName`               |
| `doXYZ()`   | Bound to URL endpoint          | `/.../xyz`                                   |
| `getXYZ()`  | Usually bound as a property    | `/.../xyz/` (sometimes), or `api/json` field |

## EnvVar

## Extension Annotation

- How it works?
- How inverse injection works? Comparison to python `__init_subclass__`.
- How Symbol annotation work

## Script Console

## Agent

### Agent parser

### Pipeline execution

1. It is a groovy script
2. `agent {}` is actually a function call. It is a great disguise as a DSL.
3. so how the functions are called? where defined? what arguments?

```groovy
import org.jenkinsci.plugins.pipeline.modeldefinition.agent.DeclarativeAgentDescriptor
DeclarativeAgentDescriptor.all().each { descriptor -> println descriptor.name }
```

The result is

```
docker
dockerfile
kubernetes
dockerContainer
label
any
none
```

## Step descriptor

```groovy
import org.jenkinsci.plugins.workflow.steps.StepDescriptor
def stepNames = StepDescriptor.all().collect { it.functionName }.sort()
stepNames.each { println it }
println "\nTotal steps: ${stepNames.size()}"
```

The result list is long, so I omit them. You could found `sh`, `sleep` etc show
up in the list.

## Plugin/Package Index

<table>
  <tr>
    <td><b>Package</b></td>
    <td><b>Depends On</b></td>
    <td><b>Purpose</b></td>
    <td><b>Notes</b></td>
  </tr>
  <tr>
    <td> jenkins-core </td>
    <td></td>
    <td> Define core concepts such as Job, Run, Action, Extension, etc.</td>
    <td></td>
  </tr>
  <tr>
    <td> jenkins-war </td>
    <td></td>
    <td>
        Start the Jenkins controller and web app, so user can interact with the UI. <br>
        It uses Stapler web framework, <br>
        which exports data as a URL (e.g., getting job info as JSON or XML via <br>
        an endpoint like /job/my-job/api/json).
    </td>
    <td>This package exist in the same repo as jenkins-core.</td>
  </tr>
  <tr>
    <td> workflow-job-plugin </td>
    <td> workflow-api-plugin </td>
    <td>
        Define pipeline job type.
    </td>
    <td/>
  </tr>
  <tr>
    <td> workflow-cps-plugin </td>
    <td> workflow-job-plugin </td>
    <td>
        Pipeline Execution Engine	
    </td>
    <td/>
  </tr>
  <tr>
    <td> pipeline-model-definition-plugin </td>
    <td> workflow-cps-plugin </td>
    <td>
        Declarative Syntax Parser
    </td>
    <td/>
  </tr>
</table>
