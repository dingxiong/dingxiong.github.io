---
layout: post
title: Jenkins -- Agent
date: 2024-10-30 18:24 -0700
categories: [devops, cicd]
tags: [devops, jenkins]
---

Jenkins has a master-agent architecture. Agent, i.e., worker is also called
"node" in Jenkins terminology. Master distributes jobs to agents.

## Agent vs inbound agent

You may wonder why there are two agent docker images:
[jenkins/agent](https://hub.docker.com/r/jenkins/agent) and
[jenkins/inbound-agent](https://hub.docker.com/r/jenkins/inbound-agent). What
are their differences? This naming conversion comes from the two different
communication methods between master and agent. One method is that master
launches agents as it needs. The other method is the reverse: the agent
initializes a connection to master so master knows a new agent is up running.
Both methods are implemented in the
[Jenkins remoting](https://github.com/jenkinsci/remoting) library. So basically
agent and inbound-agent have the same underlying jar. I know there are so many
Jenkins repos which make the relationship hard to identify. Let me explain a
little bit more. Though agent and inbound-agent docker images are the same
thing. But there are two repos that build them separately:
[jenkinsci/docker-agent](https://github.com/jenkinsci/docker-agent) and
[jenkinsci/docker-inbound-agent](https://github.com/jenkinsci/docker-inbound-agent).
So you see these Jenkins guys put dockerfiles into different repos instead of
the source code repo. What the fuck!

BTW, you can compare the
[debain agent Dockerfile](https://github.com/jenkinsci/docker-agent/blob/master/debian/Dockerfile)
with
[debain inbound agent Dockerfile](https://github.com/jenkinsci/docker-inbound-agent/blob/master/debian/Dockerfile).
You see that the latter is a replicate of the former.

Let's look at a real-world example. Below is the description of containers
inside a Jenkins k8s agent.

```
Containers:
  dind-daemon:
    Container ID:   containerd://1eb2de84b871c77e15fe3954078d89a28e6ec12a240bd932a3aeedeaabff18a0
    Image:          docker:18.09.8-dind
    Image ID:       docker.io/library/docker@sha256:8a56861f149092e7376bf672e8799332c1dd7fbbe2616cd8dfdc83152dcb52dd
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 13 Jul 2023 16:44:53 -0700G
    Ready:          True
    Restart Count:  0
    Limits:
      memory:  8704Mi
    Requests:
      cpu:        1500m
      memory:     8Gi
    Environment:  <none>
    Mounts:
      /home/jenkins/agent from workspace-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-74sh9 (ro)
      /var/staticbuild from staticbuild (rw)
  jnlp:
    Container ID:   containerd://6c756b797a0c6d5d1921a5f52246a4b4310da0dbc351394043750681fce50f62
    Image:          jenkins/inbound-agent:4.3-4
    Image ID:       docker.io/jenkins/inbound-agent@sha256:62f48a12d41e02e557ee9f7e4ffa82c77925b817ec791c8da5f431213abc2828
    Port:           <none>
    Host Port:      <none>
    State:          Running
      Started:      Thu, 13 Jul 2023 16:45:07 -0700
    Ready:          True
    Restart Count:  0
    Requests:
      cpu:     100m
      memory:  256Mi
    Environment:
      JENKINS_SECRET:         0b2e90c1f59c4c83cb9e9c0d4260ae5a8532f7b78afb4ff8f084e7d2362a7c74
      JENKINS_TUNNEL:         jenkins-agent:50000
      JENKINS_AGENT_NAME:     diffy-candidate-145-lczhf-t4518-5lxm9
      JENKINS_NAME:           diffy-candidate-145-lczhf-t4518-5lxm9
      JENKINS_AGENT_WORKDIR:  /home/jenkins/agent
      JENKINS_URL:            http://jenkins:8080/
    Mounts:
      /home/jenkins/agent from workspace-volume (rw)
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-74sh9 (ro)
```

The `jnlp` container is auto created by Jenkins k8s plugin. You can see that it
runs an inbound-agent image. Let's see what runs inside it:

```
UID        PID  PPID  C STIME TTY          TIME CMD
jenkins      1     0  4 Jul13 ?        00:00:50 /usr/local/openjdk-8/bin/java -cp /usr/share/jenkins/agent.jar hudson.remoting.jnlp.Main -headless -tunnel jenkins-agent:50000 -url http://jenkins:8080/ -workDir /home/jenkins/agent 0b2e90c1f59c4c83cb9e9c0d4260ae5a8532f7b78afb4ff8f084e7d2362a7c74 diffy-candidate-145-lczhf-t4518-5lxm9
```

I haven't read the source code, but I guess `-tunnel jenkins-agent:50000` means
agent uses this url:port to talk to master. Let's check what services are
available in the same namespace.

```
$ k get svc
NAME               TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)     AGE
jenkins            ClusterIP   10.100.1.39      <none>        8080/TCP    2y251d
jenkins-agent      ClusterIP   10.100.67.40     <none>        50000/TCP   2y251d
```
