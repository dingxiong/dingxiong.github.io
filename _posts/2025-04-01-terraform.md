---
layout: post
title: Terraform
date: 2025-04-01 18:23 -0700
categories: [devops, cicd]
tags: [devops, terraform]
---

Building Terraform from source code is simple: `go build`. What is annoying is
that we must execute it in folder with a `*.tf`. This conflicts with `delve`
because the later needs to run inside Terraform source code folder, so I could
not debug Terraform! I must missed some tricks somewhere. It should not be so
stupid.

All Terraform commands can be found
[here](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/commands.go#L388).

## Terraform Core Components

### Backend

A backend is a place to store the desired cluster state. It is used during
`terraform plan/apply` to figure out the drift and changes to apply. Terraform
supports many backends ranging from lock file to various cloud providers. Since
I have only used S3 backend, this section only covers S3 backend. All relevant
code is inside
[this folder](https://github.com/hashicorp/terraform/tree/5c4c6698822424793508642701185c04f32b05a4/internal/backend/remote-state/s3).
A sample S3 backend configuration is as follows,

```
$ cat .terraform/terraform.tfstate
{
  "version": 3,
  "terraform_version": "1.11.3",
  "backend": {
    "type": "s3",
    "config": {
      "access_key": null,
      "acl": null,
      "allowed_account_ids": null,
      "assume_role": null,
      "assume_role_with_web_identity": null,
      "bucket": "zip-prod-cdktf-state",
      "custom_ca_bundle": null,
      "dynamodb_endpoint": null,
      "dynamodb_table": "zip-cdktf-state_lockid",
      "ec2_metadata_service_endpoint": null,
      "ec2_metadata_service_endpoint_mode": null,
      "encrypt": true,
      "endpoint": null,
      "endpoints": null,
      "forbidden_account_ids": null,
      "force_path_style": null,
      "http_proxy": null,
      "https_proxy": null,
      "iam_endpoint": null,
      "insecure": null,
      "key": "cdktf.tfstate",
      "kms_key_id": null,
      "max_retries": null,
      "no_proxy": null,
      "profile": null,
      "region": "us-east-2",
      "retry_mode": null,
      "secret_key": null,
      "shared_config_files": null,
      "shared_credentials_file": null,
      "shared_credentials_files": null,
      "skip_credentials_validation": null,
      "skip_metadata_api_check": null,
      "skip_region_validation": null,
      "skip_requesting_account_id": null,
      "skip_s3_checksum": null,
      "sse_customer_key": null,
      "sts_endpoint": null,
      "sts_region": null,
      "token": null,
      "use_dualstack_endpoint": null,
      "use_fips_endpoint": null,
      "use_lockfile": null,
      "use_path_style": null,
      "workspace_key_prefix": null
    },
    "hash": 982998060
  }
}
```

There are few commands to read the terraform state. `terraform state list`
shows the tracked item names. Terraform calls them "addresses". If you think of
a state as a hash map. These `addresses` are the map keys. Each address
uniquely identifies a resource. `terraform state show <address>` shows the
specific address in detail, i.e., retrieving value for this key.
`terraform state pull` dumps the remote state as terminal output in Json
format. For example, if the `terraform.tfstate` file is stored in s3, then this
command just dump its content as stdout in the current terminal.

Here are some more details about address naming convention. And address
contains a module and a resource. If the module part is absent, then it belongs
to the global module. Module instance has
[name rule](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/addrs/module_instance.go#L272).
For example `module.app.module.db`. In construct, the name for a resource has
[name rule](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/addrs/resource.go#L24).
So an absolute resource has
[name](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/addrs/resource.go#L222).
For example `module.app.module.db.aws_db_instance.db`.

There are a few ways of updating the state file. `terraform apply` makes
infrastructure changes together with updating the state file.
`terraform state push`, on the other hand, replacing remote state file with a
local one. It is dangerous and should only be used for disaster recovery or
importing large chunk of existing infrastructure.

How to get the state file ?

How state file is parsed?

https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/states/statefile/read.go#L90

### Import State

In order to bring existing infrastructure to Terraform, we need to do one-time
effort of including it to the terraform state. This can be down by
`terraform import` or `terraform plan/apply`. The syntax is

```
terraform import [options] ADDR ID

Ex: terraform import aws_instance.example i-abcd1234
```

For terraform plan/apply, we should add an import section to the .tf file. For
example,

```
provider "aws" {
  region = "us-east-1"
}

resource "aws_s3_bucket" "terrateam-block-import" {
}

import {
  id = "terraform-import-block-bucket"
  to = aws_s3_bucket.terrateam-block-import
}
```

The second approach is safer because `terraform plan` is 100% safe, so let's
lee what happens for this import clause. First, it will be
[parsed](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/terraform/context_plan.go#L640)
to be used for validation and planing purpose. Second, during planing stage,
the import id will be used to fetch the configurations of existing resources
matching this id. Depending on the resource type, the meaning of id is
different. For AWS EC2, the id the EC2 id. For AWS RDS cluster, the id is the
cluster name. A provider
[proto function](https://github.com/hashicorp/terraform-plugin-go/blob/ee6e52ba66ed669583b507f13068b818bce39614/tfprotov6/internal/tfplugin6/tfplugin6_grpc.pb.go#L371)
is reserved for this purpose. It is called at
[here](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/terraform/node_resource_plan_instance.go#L652)

### How does Plan Walk the Resource Tree?

TDB...

```
terraform/internal/terraform/node_resource_plan.go:562 +0x5b8
terraform/internal/terraform/transform_resource_count.go:32 +0x3b8
terraform/internal/terraform/graph_builder.go:47 +0x160
terraform/internal/terraform/node_resource_plan.go:508 +0x4c4
terraform/internal/terraform/node_resource_plan.go:445 +0x2f8
terraform/internal/terraform/node_resource_plan.go:363 +0x218
terraform/internal/terraform/node_resource_plan.go:140 +0x460
terraform/internal/terraform/graph.go:153 +0x770
terraform/internal/dag/walk.go:393 +0x2a8
terraform/internal/dag/walk.go:316 +0xc44
```

### State Lock

A few commands such as `state push`, `plan` and `apply` will acquire a
distributive lock for the remote state.

S3 backend uses DynamoDB's conditional write mechanism to implement a
distributed lock. There are two records in the table.

```
LockID                      | Digest
----------------------------| -----
state-s3-path               | <empty>
state-s3-path-<md5-suffix>  | xxxxx..
```

The first record is the lock. Usually, it is not there because it only exits
between `lock` and `unlock` which is an transient state.

The second record is the md5 hash of the state content. When the state is
updated, the md5 is updated at the same time. See
[code](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/backend/remote-state/s3/client.go#L240).

### Logging

[TF_LOG](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/logging/logging.go#L20)
controls the log level. If we do not set it, then it means
[no log](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/logging/logging.go#L185).
An example:

```
AWS_PROFILE=admin TF_LOG=INFO ~/tmp/terraform/terraform plan
```

One interesting thing is that Terraform uses the golang std log library, not a
third party logging framework. To achieve this flexibility and simplicity, It
changes the output destination for the std logger. See
[code](https://github.com/hashicorp/terraform/blob/5c4c6698822424793508642701185c04f32b05a4/internal/logging/logging.go#L55).

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
