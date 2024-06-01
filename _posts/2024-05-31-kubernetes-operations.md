---
layout: post
title: Kubernetes -- Operations
date: 2024-05-31 21:06 -0700
categories: [kubernetes]
tags: [kubernetes]
---

## Operatons

### Increase the disk size of StatefulSet PVC

```bash
kubectl edit pvc <pvc-name> -n <namespace>
kubectl delete statefulset --cascade=orphan <statefulset_name> -n <namespace>
# Then you can reapply the helm chart
```

## Pod

I learned a lot from Lan Lewis's blog
[Almighty pause container](https://www.ianlewis.org/en/almighty-pause-container)
and
[what are kubernetes pods anyway?](https://www.ianlewis.org/en/what-are-kubernetes-pods-anyway).
Basically, containers in a pod shared the same PID and NETWORK linux namespace,
They are all child processes of a special
[`pause` container](https://github.com/kubernetes/kubernetes/blob/master/build/pause/linux/pause.c).
Therefore, containers belong to the same pod can talk to each other just using
`localhost`. They can also share the same PID namespace given
`shareProcessNamespace: true` in the pod configuration.

This makes me think whether containers are really isolated with each other.

However, containers in the same pod do not share the cgroup. For example:

```
$ systemd-cgls memory
...

  │ ├─kubepods-burstable-poda098f231_8c0e_40b9_a996_28807188763e.slice
  │ │ ├─cri-containerd-c7b23483797bbbe96f3bebfeac360a6c69518649790b93f3056b4061cfdf756e.scope
  │ │ │ └─18227 sleep infinity
  │ │ ├─cri-containerd-667351b7b29abb99946977a36f49e7e3cb5a28d4f6d994260fe52d6b0f90125a.scope
  │ │ │ └─18137 java -Djdk.httpclient.allowRestrictedHeaders=host,connection,content-length,expect,upgrade -Dsun.net.httpserver.maxRspTime=60 -jar /diffy.jar --candidate=server.production-diffy-candidate.svc.cluster.local:8000 --master.primary=server.production-dif
  │ │ └─cri-containerd-0e4ff4807ce331937571acd3c8bcc30bc19c067877433b947d7745885a70cfbd.scope
  │ │   └─18057 /pause
...
```

You can see the above pod has three containers which running commands:

- sleep infinity
- java -D... diffy.jar ...
- pause

This means that the `resources.limit` fields are at container level, not pod
level. Namely, each container con configure its own cpu/memory resources.
