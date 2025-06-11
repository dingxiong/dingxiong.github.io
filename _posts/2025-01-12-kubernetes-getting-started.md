---
layout: post
title: Kubernetes -- Getting Started
date: 2025-01-12 12:44 -0800
categories: [devops, kubernetes]
tags: [kubernetes]
---

- [cheet sheet](https://kubernetes.io/docs/reference/kubectl/cheatsheet/)

As a infra engineer, it is better to remember the json paths of some frequently
used properties.

- `.spec.nodeName` useful if you want to sort pods by nodes.

Quickly get a working pod

```
k run my-shell --rm -it --image ubuntu -- bash

k run xiong-ubuntu-test --image=ubuntu --rm=true --restart=Never -it --overrides='{ "apiVersion": "v1", "spec": {"nodeSelector": {"role": "xiong-test"}} }' -- bash

k run kafka-consumer -n staging-oneoff --image=ubuntu:latest --rm=true --restart=Never -it \
--overrides='{"spec":{"nodeSelector":{"role":"staging-job"},"tolerations":[{"effect": "NoSchedule","key": "dedicated","operator": "Equal","value": "staging-job"}]}}'  \
-- bash

```

## Raw commands

- Get kubelet config
  `k get --raw='/api/v1/nodes/ip-172-31-90-143.us-east-2.compute.internal/proxy/configz' | jq '.'`

## kustomize

1. Use `kind component` to define reusable components.
2. For `Job` kind, Do not specify `spec.selector` unless you have a strong
   reason. See
   https://kubernetes.io/docs/concepts/workloads/controllers/job/#specifying-your-own-pod-selector

```
## view the new yaml
kustomize build <folder>
or
k kustomize <folder>
```

Example:
https://stackoverflow.com/questions/71165168/can-someone-explain-patchesstrategicmerge

## node

ssh into a node

```
minikube ssh -p minikube-multi-nodes -n minikube-multi-nodes-m02
```

Then you can run `docker ps` to see what containers are running in this node.

In production env, you can install
[node-shell](https://github.com/kvaps/kubectl-node-shell) throw `krew`.

## Service

Test a service

```
EXTERNAL_IP=$(kubectl get svc nginx -o jsonpath="{.status.loadBalancer.ingress[*].ip}")
curl -s http://$EXTERNAL_IP
```

When rolling upgrade of a service is not possible. Lan Lewis provide a
blue/green deployment strategy
https://www.ianlewis.org/en/bluegreen-deployments-kubernetes.

## Mount

Checkout
[mount propagation](https://kubernetes.io/docs/concepts/storage/volumes/#mount-propagation)
to learn how to share mounts among containers.

## istio

This post helps me a lot
https://stackoverflow.com/questions/47664397/istio-ingress-resulting-in-no-healthy-upstream
for debugging istio issue.

```
istioctl proxy-config endpoints sso-proxy-65f77877fb-mfpvq.sso --cluster "outbound|8004||one-off-master.staging-oneoff.svc.cluster.local"
```

## Autoscaler

There are roughly two types of auto scaler. One that scales pods. The other
scales nodes. We are familiar with the latter one
[later one](https://github.com/kubernetes/autoscaler/tree/master/cluster-autoscaler).

Pod autoscaler is built in Kubernetes. HPA (Horizontal Pod Autoscaler) is
mostly frequently used. You can create it by `k autoscale`. The default metrics
you can depend on are CPU and memory. You can also use custom metrics and
external metrics. Most popular tools provide custom/external metrics for you to
use. For example, Prometheus has
[adaptor](https://github.com/kubernetes-sigs/prometheus-adapter) to provide
custom metrics. Datadog also has such add-ons.

### Metrics-server

[Metrics-server](https://github.com/kubernetes-sigs/metrics-server) is for auto
scaling.

You can use `k top node/pod` command to view resource usage. See above cheat
sheet.

## Controller

- CRD: custom resource definition

These posts
[1](https://vivilearns2code.github.io/k8s/2021/03/11/writing-controllers-for-kubernetes-custom-resources.html)
,
[2](https://vivilearns2code.github.io/k8s/2021/03/12/diving-into-controller-runtime.html)
are a good start to understand k8s controllers. Also, kubebuilder's official
tutorial is very useful.

## Misc

1. Certain fields are immutable, like label selectors in deployments. see
   https://kubernetes.io/docs/concepts/workloads/controllers/deployment/#label-selector-updates

# Minikube

Below is steps I set up my minikube in Mac M1.

### After installation, run below to create a cluster

```
minikube start -p minikube-multi-nodes --nodes 2 --cpus=4 --memory=4096
# below is to test taint
k taint nodes <node-name> dedicated=staging-product-private:NoSchedule
k label nodes <node-name> role=staging-product-private
```

### install nginx ingress controller

Follow https://kubernetes.github.io/ingress-nginx/deploy/#quick-start

Ingress controller is itself a k8s service. So we can port forward ingress
controller to test whether it works or not. Usually, for each service that need
to be exposed to outer world, we should have ingress setup in its namespace.
Then I am wondering how ingress controller combines these ingress rules from
all namespaces. Actually, it works well. See
https://vincentlauzon.com/2020/02/11/ingress-rules-in-different-kubernetes-namespaces
You can check the Nginx config

```
ke -n ingress-nginx exec <pod-name> -- cat /etc/nginx/nginx.conf
```

## Plugins

Use plugin manager `Krew`. Useful plugins:

- node-shell: allow you to open a shell in node. So you do not need to set up
  ssh for a EC2 instance.
- k ssh: allow you ssh into a pod as root.
  [See here](https://github.com/jordanwilson230/kubectl-plugins)
