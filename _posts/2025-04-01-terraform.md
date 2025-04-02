---
layout: post
title: Terraform
date: 2025-04-01 18:23 -0700
categories: [devops]
tags: [devops, terraform]
---

## cdktf

[Terraform-cdk](https://github.com/hashicorp/terraform-cdk) is really a crappy
code base. This is my first time seeing cli logic calls react. See
[one example](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/cdktf-cli/src/bin/cmds/handlers.ts#L470).
`cdktf` has
[tree main packages](https://github.com/hashicorp/terraform-cdk/tree/v0.21.0-pre.158/packages):
`@cdktf`, `cdktf-cli` and `cdktf`. The first two are related to the cdktf cli.
The last one implements the core cdk logic.

Let's see what happens when running `cdktf synth`. The call path is

1. [cdktf-cli/src/bin/cmds/synth.ts](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/cdktf-cli/src/bin/cmds/synth.ts#L48)
2. [cmds/handlers.ts:synth](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/cdktf-cli/src/bin/cmds/handlers.ts#L445)
3. [cmds/ui/synth.tsx](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/cdktf-cli/src/bin/cmds/ui/synth.tsx#L34)
4. [@cdktf/cli-core/src/lib/cdktf-project.ts](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/@cdktf/cli-core/src/lib/cdktf-project.ts#L358)
5. [@cdktf/cli-core/src/lib/synth-stack.ts](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/@cdktf/cli-core/src/lib/synth-stack.ts#L74)

In the second step above, we check whether the current repo is a valid cdktf
repo through function `throwIfNotProjectDirectory()`. Basically, it checks
whether the current directory has file `cdktf.json` or not, and this file
contains a json object which contains two fields: `language` and `app`. For
example

```
$ cat cdktf.json
{
  "app": "pipenv run python main.py",
  "language": "python",
  ...
}
```

This tells cdktf-cli that the programming language used to define terraform
structs and how we can run this project. See
[code](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/cdktf-cli/src/bin/cmds/handlers.ts#L456).
In step 5 above, it launch a
[shell](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/@cdktf/cli-core/src/lib/synth-stack.ts#L128)
to run this command. Therefore, `cdktf-cli` and `@cdktf` are drivers, they do
not contain Terraform stack related logic. A simple stack that creates a s3
bucket looks like code below.

```python
from cdktf import App, TerraformStack
from cdktf_cdktf_provider_aws.s3_bucket import S3Bucket
from constructs import Construct

class MyStack(TerraformStack):
    def __init__(self, scope: Construct, id: str):
        super().__init__(scope, id)

        S3Bucket(self, bucket_name="xx")

app = App()
MyStack(app, "my-stack")
app.synth()
```

We create a terraform stack `MyStack`, and a scope `app`. A scope is like a
namespace which holds a tree of constructs. The tree can be accessed from the
tree root node. See
[code](https://github.com/aws/constructs/blob/5bcb1e0ca14539266b14c9413649c2e7aea4df9e/src/construct.ts#L471C1-L472C1).
At line `S3Bucket(self, bucket_name="xx")`, we register an S3 bucket as a
[child node of this tree](https://github.com/aws/constructs/blob/5bcb1e0ca14539266b14c9413649c2e7aea4df9e/src/construct.ts#L71).
The `app.synth()` step is simple. It walks through the tree to generate the
terraform definition file. See
[code](https://github.com/hashicorp/terraform-cdk/blob/071692f66da8478d9a76165bba6ec1b484ec8c78/packages/cdktf/lib/app.ts#L149).
You can read more about the aws constructs github repo if you are interested in
the details.
