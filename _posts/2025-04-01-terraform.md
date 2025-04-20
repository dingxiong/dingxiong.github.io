---
layout: post
title: Terraform
date: 2025-04-01 18:23 -0700
categories: [devops]
tags: [devops, terraform]
---

Building Terraform from source code is simple: `go build`. What is annoying is
that we must execute it in folder with a `*.tf`. This conflicts with `delve`
because the later needs to run inside Terraform source code folder, so I could
not debug Terraform! I must missed some tricks somewhere. It should not be so
stupid.

All Terraform commands can be found
[here](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/commands.go#L388).

Q1: how does dynamodb lockfile work?

## Terraform Core Components

### Backend

Interesting to see how module instance get its name
[name](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/addrs/module_instance.go#L272).
For example `module.app.module.db`.

In construct, the name for a resource is
[here](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/addrs/resource.go#L24).

Absolute resource has
[name](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/addrs/resource.go#L222).

For example `module.app.module.db.aws_db_instance.db`.

How to get the state file ?

How state file is parsed?

https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/states/statefile/read.go#L90

`terraform state list` and `terraform state show` tells the backend state.

`terraform state pull`: dump the remote state as terminal output in Json
format. For example, if the `terraform.tfstate` file is stored in s3, then this
command just dump its content as stdout in the current terminal.

`terraform state push`: This is dangerous. It pushes local state file to
remote.

## Terraform plugin

Terraform provides a go library `terraform-plugin-framework` to help people
write 3rd party plugins, and it provides a
[official tutorial](https://developer.hashicorp.com/terraform/tutorials/providers-plugin-framework).
The core part is a set of
[proto contracts](https://github.com/hashicorp/terraform-plugin-go/blob/ee6e52ba66ed669583b507f13068b818bce39614/tfprotov6/internal/tfplugin6/tfplugin6_grpc.pb.go#L345).
However, Terraform team makes layers of abstractions on top these proto
definitions, so it is not obvious the first time you read this code and wonder
how they are hooked up. For example,
[tf6server.server](https://github.com/hashicorp/terraform-plugin-go/blob/ee6e52ba66ed669583b507f13068b818bce39614/tfprotov6/tf6server/server.go#L408)
implements the proto `ProviderServer` interface, but it delegates all implement
details to its member variable `downstream tfprotov6.ProviderServer`. Note this
`ProviderServer` interface is not the proto `ProviderServer` interface. In this
way, it detaches the contract interface from the proto definitions, so
`terraform-plugin-go` has two layers of abstractions. Users only need to worry
about `tfprotov6.ProviderServer` but not the original proto definitions.

This is not the end of the story. Let's see what happens on the side of
`terraform-plugin-framework`.
[proto6server.server](https://github.com/hashicorp/terraform-plugin-framework/blob/f4f3ad9b7c7c11729980fa658d77c3a69754ffe6/internal/proto6server/serve.go#L14)
implements `tfprotov6.ProviderServer` interface. However, all implementations
are delegated to its member variable `FrameworkServer fwserver.Server`. For
example, you can find the `ReadDataSource` implementation
[here](https://github.com/hashicorp/terraform-plugin-framework/blob/f4f3ad9b7c7c11729980fa658d77c3a69754ffe6/internal/fwserver/server_readdatasource.go#L38).
As a user, I only need to implement the `DataSource` interface. Now, you
understand this whole shit layers of abstractions. They must be Java
programmers!

One nit detail. Popular plugins such as `terraform-plugin-aws` uses
[terraform-plugin-sdk](https://github.com/hashicorp/terraform-plugin-sdk)
instead of `terraform-plugin-framework`. This sdk library is about to
deprecated. Newer plugins should build against the plugin framework.

## Terraformer

This is a typical Cobra cmd line application. For AWS, it uses `aws-sdk-go` to
load the configurations of various resources and dump them into terraform
configuration files. See
[RDS example](https://github.com/GoogleCloudPlatform/terraformer/blob/698f2d52547f67015e44932b6568fc42d5484d3a/providers/aws/rds.go#L32).

It also requires you have the corresponding terraform plugin
[pre-installed](https://github.com/GoogleCloudPlatform/terraformer/blob/698f2d52547f67015e44932b6568fc42d5484d3a/terraformutils/providerwrapper/provider.go#L273).
This is used during terraform configuration file generation stage. It is not
used for fetching resource definitions. It first tries to load the plugin in
the current `.terraform` folder. If not found, then go to the global plugin
cache folder `$HOME/.terraform.d/plugins`. The annoying thing about terraform
is that it does not provide a way to download a plugin directly. You must
create a directory and put a `*.tf` file inside this folder like the one below
and run `terraform init` to download the plugin indirectly.

```
terraform {
  required_providers {
    aws = {
      source = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region = "us-east-1"
}
```

Then run `terraformer` inside this directory.

One additional note about filters: according to the documentation, we can use
filters to narrow down the resource we want to import. For example,

```
terraformer import aws -r rds --profile=admin -f Type=sg;Name=vpc_id;Value=VPC_ID
```

However, I find only a few small subset of resources support it. For example,
[AWS EC2](https://github.com/GoogleCloudPlatform/terraformer/blob/698f2d52547f67015e44932b6568fc42d5484d3a/providers/aws/ec2.go#L41)
supports it. On the other hand,
[AWS RDS](https://github.com/GoogleCloudPlatform/terraformer/blob/698f2d52547f67015e44932b6568fc42d5484d3a/providers/aws/rds.go#L258)
does not support it.

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
