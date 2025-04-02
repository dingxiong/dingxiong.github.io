---
title: Spinnaker overview
date: 2023-12-08 10:01 -0800
categories: [devops]
tags: [devops, spinnaker]
---

Spinnaker different components port number table:
[link](https://spinnaker.io/docs/reference/architecture/microservices-overview/)

## Pipeline

I was very confused by the expression evaluation syntax used in UI in the
beginning. Later on, I realized that it is just
[SpEL (Spring Expression Language)](https://docs.spring.io/spring-framework/reference/core/expressions/evaluation.html).
So an expression `${ a == 'xxx'}` stands for a condition that tests whether the
content of variable `a` equals the string `xxx`.

Then the question is how it looks up variable `a` and why not treat it as a
literal? To answer this question, we need go deep into SpEL. This
[demo](https://github.com/dingxiong/codebox/blob/55a76a2e6d975b0a654184d38cac9fc31e70a5a5/SpELDemo/src/test/java/com/example/SpELTest.java#L50-L50)
has a few examples. There are two ways to reference a variable:

1. Set it to the context and retrieve it by `#var_name`. This is called global
   variable.
2. Set the variable as a member of the root object, and evaluate the expression
   against the root object, then `var_name` is automatically evaluated to its
   value. In this case, `#` is not needed to prefix `var_name`.

Spinnaker chooses the second approach. It passes the stage context as the root
object to the evaluation function, but the stage context is a `Map`, then how
variables are resolved? It provides some
[customized property accessors](https://github.com/spinnaker/kork/blob/cf76e731d4201d0972cec6197a0f2536046e4537/kork-expressions/src/main/java/com/netflix/spinnaker/kork/expressions/ExpressionsSupport.java#L177)
to help SpEL resolve this property.

Another note about `Evaluate Variables` stage is that if we define multiple
variables in this stage, then latter variable evaluation can use previous
variables' evaluation result. See code
[here](https://github.com/spinnaker/orca/blob/0284e7537af855159da2511aae2e0f89d8910342/orca-core/src/main/java/com/netflix/spinnaker/orca/pipeline/EvaluateVariablesStage.java#L103-L103)
that it puts the intermediate evaluation result back to the augmented context.

Let's go deeper into SpEL (maybe I should open a new note for it). How is an
expression evaluated? Most time we use
[TemplateParserContext](https://github.com/spring-projects/spring-framework/blob/a3b979c5ecb7eda96ebf264672ce522177c6fc77/spring-expression/src/main/java/org/springframework/expression/common/TemplateParserContext.java#L38)
so the expressions are enclosed inside a `${`, `$}` pair. Expressions are first
parsed and then
[evaluated](https://github.com/spinnaker/kork/blob/cf76e731d4201d0972cec6197a0f2536046e4537/kork-expressions/src/main/java/com/netflix/spinnaker/kork/expressions/ExpressionTransform.java#L177).
This `getValue` function is where the interesting story starts. This method is
an abstract method in class
[CompiledExpression](https://github.com/spring-projects/spring-framework/blob/a3b979c5ecb7eda96ebf264672ce522177c6fc77/spring-expression/src/main/java/org/springframework/expression/spel/CompiledExpression.java).
But nobody inherits this class and nobody explicitly provides an implementation
for it! All its implementations are indeeded constructed on the fly in
[SpelCompiler](https://github.com/spring-projects/spring-framework/blob/a3b979c5ecb7eda96ebf264672ce522177c6fc77/spring-expression/src/main/java/org/springframework/expression/spel/standard/SpelCompiler.java#L137).

Does space matter? This is the another confusion I have with Spinnaker. In
Spinnaker's official documentations, it always adds spaces between the
expression to be evaluated such as `${ xxxx }`. Is this necessary. Actually
not. SpEL trims the component. See
[code](https://github.com/spring-projects/spring-framework/blob/a3b979c5ecb7eda96ebf264672ce522177c6fc77/spring-expression/src/main/java/org/springframework/expression/common/TemplateAwareExpressionParser.java#L122)

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
