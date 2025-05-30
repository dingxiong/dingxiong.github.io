---
layout: post
title: AWS -- Getting Started
date: 2025-01-12 12:34 -0800
categories: [aws]
tags: [aws]
---

## AWS load balancer controller

An example ALB created by controller.
`k8s-albnginxingress-119b4b671c-832110419.us-east-2.elb.amazonaws.com` The name
comes from
[here](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/c94ffd3055892235c77a4024817367207bc6f94c/pkg/ingress/model_build_load_balancer.go#L108)

The reconcile step is
[here](https://github.com/kubernetes-sigs/aws-load-balancer-controller/blob/c94ffd3055892235c77a4024817367207bc6f94c/controllers/ingress/group_controller.go#L120).

## EC2

How to ssh to a node:

```
ssh -i infra.pem ec2-user@ip-172-31-64-166.us-east-2.compute.internal
```

## IMDS

IMDS (Instance Metadata Service) is a service that provides metadata and user
data to EC2 instances, such as instance type, instance id, image id, IAM role,
etc. This information is mainly used for secured authentication. Applications
running on EC2 instances must sign their requests with AWS credentials. One way
is to provide the credentials in the form of AWS access keys/secrets. But to be
safe, you need to rotate them periodically. Both the security and infra team
hate this manual work! The other approach is authenticating through IAM roles.
This is more popular. In EKS, this means you run the application using a
service account which is attached with an IAM role. What happens underneath is

- Application sends a request
  `http://169.254.169.254/latest/meta-data/iam/security-credentials/` to get
  the associated IAM role.
- Then it sends another request
  `http://169.254.169.254/latest/meta-data/iam/security-credentials/<iam_role_name>/`
  to get a temporary access credential.

IP `169.254.169.254` is a hard-coded private IP that is injected by AWS to
every EC2's route table. It is guaranteed to be there! You can see some
examples in my Kubernetes network blog.

IMDS is also used by some observability tools such as datadog to label
instance. Checkout out my datadog blogs about how it is used.

IMDS itself has some security loopholes. So this leads to invention of IMDSv2.
All requests to IMDSv2 should attach a session token, so the first step is to
obtain a session token by the following call.

```
curl -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600"
```

The `ttl` header is required, and the client side is responsible for rotating
this session token before it expires. Luckily, most aws client libraries has
already done it for us.

## EKSCTL

`eksctl` is a command line tool to manage AWS EKS cluster. Its github front
page says it is "The official CLI for Amazon EKS", but AWS support does not
agree with it. :)

No software is bug-free, including this tool. See
[how many issues this too had/has](https://github.com/weaveworks/eksctl/issues)!
This post is a study note of how eksctl works internally. We need to understand
it, otherwise ignorance of a bug in it will create disastrous incident.

> I was hit by a bug in eksctl that led ziphq.com down for 15min!

### Delete node group

The official documentation says you can delete a node group by below command

```
eksctl delete nodegroup --cluster=product --name=<...>
```

However, there are a few nuances.

First, this command will delete the IAM role in the `aws-auth` config map by
default. It other node groups use the same IAM role, then you are fucked up!
The other node group will be let hard down. See
[this issue](https://github.com/weaveworks/eksctl/issues/5690). Fortunately,
there is a flag to disable this step:

```
eksctl delete nodegroup --cluster=product --name=<...> --update-auth-configmap=false
```

Eksctl's official documentation does not mention this parameter at all! You can
see the code
[here](https://github.com/weaveworks/eksctl/blob/5746832673792da6dba9ab892d9cda9ce6a0efad/pkg/ctl/delete/nodegroup.go#L160-L161).

Second, this command treats unmanaged node group and managed node group
differently. The core logic of deleting a node group is
[here](https://github.com/weaveworks/eksctl/blob/5746832673792da6dba9ab892d9cda9ce6a0efad/pkg/actions/nodegroup/delete.go#L14).
You see this function takes arguments `nodeGroups` and `managedNodeGroups`.
`nodeGroups` represents unmanaged node groups. All unmanaged node groups have a
CloudFormation stack associated with it, but a managed node group may or may
not be associated with a CloudFormation stack. eksctl uses
[CloudFormation tags](https://github.com/weaveworks/eksctl/blob/5746832673792da6dba9ab892d9cda9ce6a0efad/pkg/cfn/manager/nodegroup.go#L162)
to determine whether if it is managing a node group. Use command below to check
if a stack is a node group manager

```
aws cloudformation describe-stacks | jq -C ".Stacks[] | {id: .StackId, tags: [.Tags]}"
```

If a managed node group is associated with a stack, then it is deleted the same
way as unmanaged node group. If not, then eksctl simply calls
[aws eks delete-nodegroup api](https://github.com/weaveworks/eksctl/blob/5746832673792da6dba9ab892d9cda9ce6a0efad/pkg/cfn/manager/delete_tasks.go#L140),
which is equivalent to deleting node group in aws eks dashboard.

Blow is an example of successful unmanaged node group deletion

```
$ eksctl delete nodegroup --cluster=product ng-staging-product-private  --profile=admin --update-auth-configmap=false
2023-02-21 14:07:18 [ℹ]  1 nodegroup (ng-staging-product-private) was included (based on the include/exclude rules)
2023-02-21 14:07:18 [ℹ]  will drain 1 nodegroup(s) in cluster "product"
2023-02-21 14:07:18 [ℹ]  starting parallel draining, max in-flight of 1
2023-02-21 14:07:18 [!]  no nodes found in nodegroup "ng-staging-product-private" (label selector: "alpha.eksctl.io/nodegroup-name=ng-staging-product-private")
2023-02-21 14:07:18 [ℹ]  will delete 1 nodegroups from cluster "product"
2023-02-21 14:07:19 [ℹ]  1 task: { 1 task: { delete nodegroup "ng-staging-product-private" [async] } }
2023-02-21 14:07:20 [ℹ]  will delete stack "eksctl-product-nodegroup-ng-staging-product-private"
2023-02-21 14:07:20 [✔]  deleted 1 nodegroup(s) from cluster "product"
```
