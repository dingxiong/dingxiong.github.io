---
title: Spinnaker overview
date: 2023-12-08 10:01 -0800
categories: [spinnaker]
tags: [spinnaker]
---

Spinnaker different components port number table:
[link](https://spinnaker.io/docs/reference/architecture/microservices-overview/)

## Cloud driver

Cloud driver starts an docker registry agent, that pull image tags to cache
(most likely Redis cache) every 30 seconds.

How to check the image tags in the docker registry?

Use an endpoint

```
ke spin-clouddriver-7dcfd6d5cb-xp9n2 -- bash -c 'curl localhost:7002/dockerRegistry/images/find?q=Build440' | jq .
[
  {
    "account": "my-ecr-registry",
    "artifact": {
      "metadata": {
        "labels": {},
        "registry": "242230929264.dkr.ecr.us-east-2.amazonaws.com"
      },
      "name": "evergreen-server",
      "reference": "242230929264.dkr.ecr.us-east-2.amazonaws.com/evergreen-server:Build440-2022_10_06-05_14_48-HEAD-db47ab910-Atif_Niyaz",
      "type": "docker",
      "version": "Build440-2022_10_06-05_14_48-HEAD-db47ab910-Atif_Niyaz"
    },
    "registry": "242230929264.dkr.ecr.us-east-2.amazonaws.com",
    "repository": "evergreen-server",
    "tag": "Build440-2022_10_06-05_14_48-HEAD-db47ab910-Atif_Niyaz"
  }
]
```

Just check the redis content

```
$ redis-cli -a password
127.0.0.1:6379> keys *taggedImage*my-ecr-registry*Build440*
1) "com.netflix.spinnaker.clouddriver.docker.registry.provider.DockerRegistryProvider:taggedImage:attributes:dockerRegistry:taggedImage:my-ecr-registry:evergreen-server:Build440-2022_10_06-05_14_48-HEAD-db47ab910-Atif_Niyaz"
```

## common commands

```
curl -X GET http://localhost:8083/health
k exec spin-orca-6659fb9c74-rk9mz -- bash -c "curl -X PUT http://localhost:8083/admin/forceCancelExecution?executionId=01G7ARGX0ADTPPP6DZ8TNXTSBR"
k exec spin-orca-6659fb9c74-rk9mz -- bash -c "curl -X GET http://localhost:8083/v2/applications/server/pipelines"
```

## ORCA

### ExecutionRepository

It stores execution result. For RedisExecutionRepository, the keys are
formatted as `{executionType}:{id}` Ex:

```
127.0.0.1:6379> keys pipeline*
...
9937) "pipeline:01G5JDXZW3WH2KMX4CAS9QTAQG"
9938) "pipeline:01FZ1JVC75XGB2VN9TPZRKX6RT:stageIndex"
9939) "pipeline:01G1VKET5AGN2MWQGWESYX20EF"
9940) "pipeline:01FVH18TPQ6M5E2EH5HZRQS5HN"
```

## tips

1. When add podAnnotations, you should kill the halyard statefulset and
   services.
2.
