---
layout: "cdktf"
page_title: "Project Setup"
sidebar_current: "cdktf"
description: "Build a CDKTF application from a template or existing HCL project, and configure storage for Terraform state."
---

# Project Setup

-> **Note:** CDK for Terraform is currently in [beta](/docs/cdktf/index.html#project-maturity-and-production-readiness).

There are several ways to create a new CDK for Terraform (CDKTF) project. You can create a new application from a pre-built or a custom template, and you can also convert an existing HCL project. When you create a new project, you can store Terraform state locally or use a [remote backend](/docs/cdktf/concepts/remote-backends.html). This page discusses these setup options in more detail.

> **Hands On**: Try the [CDK for Terraform Quick Start Demo](https://learn.hashicorp.com/tutorials/terraform/cdktf-install?in=terraform/cdktf) and language-specific [Get Started Tutorials](https://learn.hashicorp.com/collections/terraform/cdktf) on HashiCorp Learn.

## Initialize Project from a Template

Use `init` with a project template to automatically create and scaffold a new CDKTF project for your chosen programming language.
Templates generate a new application with the necessary file structure for you to start defining infrastructure.

You can use a `cdktf-cli` pre-built template or a custom-built [remote template](/docs/cdktf/create-and-deploy/remote-templates.html) when you initialize a new project.

```bash
$ cdktf init --template="templateName"
```

Use these template names for the available pre-built templates:

- `typescript`
- `python`
- `c#`
- `java`
- `go` (experimental)

### Use a Local Backend

Your application needs somewhere to store [Terraform state](https://www.terraform.io/docs/language/state/index.html). Add the `--local` flag to created a scaffolded project that is pre-configured to use a [local backend](https://www.terraform.io/docs/language/settings/backends/local.html). This means all terraform operations will happen on your local machine.

```
$ cdktf init --template="typescript" --local
```

### Use Terraform Cloud as a Remote Backend

If you do not want to store Terraform state locally, you can configure your app to use Terraform Cloud as a remote backend.

Run `cdktf init` without passing the `--local` flag to prompt CDKTF to use your stored [Terraform Cloud](https://www.terraform.io/docs/cloud/index.html) credentials to create a new workspace. If you have no stored Terraform Cloud credentials, CDKTF will ask you to login.

After you login, CDKTF creates a new scaffolded project that is configured to use Terraform state with a [remote backend](https://www.terraform.io/docs/language/settings/backends/remote.html). Where the Terraform operations will happen depends on the configuration of the Terraform Cloud Workspace settings. When you create the workspace as part of the `cdktf init` command, Terraform operations will be run locally by default. You can configure the Terraform Cloud workspace to use remote operations instead (details below).

#### Terraform Cloud VCS Integration

Terraform Cloud supports [connecting to VCS providers](https://www.terraform.io/docs/cloud/vcs/index.html). To use the VCS integration:

1. Commit the synthesized Terraform config (the `cdktf.out` directory) alongside your code, so that Terraform Cloud can use it to deploy your infrastructure.
2. On the **General Settings** page of your Terraform Cloud workspace, [set the Terraform working directory](https://www.terraform.io/docs/cloud/workspaces/settings.html#terraform-working-directory) to the output directory of the stack you want to deploy. For example, use `cdktf.out/stacks/dev` if your stack is named `dev`.

   Refer to the [Stacks page](/docs/cdktf/concepts/stacks.html) for more information about using stacks to separate the state management for multiple environments in an application.

~> **Important**: The synthesized Terraform config might contain credentials or other sensitive data that was provided as input for the `cdktf` application.

#### External CI service

If you prefer to keep the generated artifacts out of your repository, use any CI (Continuous Integration) service to build and deploy them instead. The CDK for Terraform CLI supports deploying to Terraform Cloud using either the local or remote execution mode. For more information on how runs work in Terraform Cloud, see [Terraform Runs and Remote Operations](https://www.terraform.io/docs/cloud/run/index.html).

Use the [`cdktf-cli` commands](/docs/cdktf/cli-reference/commands.html) as part of your CI steps to synthesize your code and deploy your infrastructure.

```
cdktf deploy --auto-approve
```

## Project Configuration

Initializing your project with a template generates a basic project in your preferred programming language that you can customize for your use case. You can manage global configuration for your project by customizing the `cdktf.json` configuration file or the application context.

### `cdktf.json` Configuration File

The `cdktf.json` configuration file is where you can define the [providers](/docs/cdktf/concepts/providers-and-resources.html) and [modules](/docs/cdktf/concepts/modules.html) that should be added to the project and supply custom configuration settings for the application. Refer to the [cdktf.json documentation](/docs/cdktf/create-and-deploy/configuration-file.html) for more detail.

### Application Context

All of the classes in your application can access the application `context`, so it is an ideal place to store project configuration. Context becomes available to any construct in your application after you run `cdktf synth`.

You can configure context as a static value in `cdktf.json` by setting the `context` property.

```jsonc
{
  // ...
  "context": {
    "myConfig": "value"
  }
```

You can also provide context when instantiating the `App` class.

```ts
const app = new App({ context: { myConfig: "value" } });
```

The TypeScript example below uses `App` context to provide a custom tag value to an AWS EC2 instance.

```typescript
import { Construct } from "constructs";
import { App, TerraformStack } from "cdktf";
import { AwsProvider, EC2 } from "./.gen/providers/aws";

class MyStack extends TerraformStack {
  constructor(scope: Construct, id: string) {
    super(scope, id);

    new AwsProvider(this, "aws", {
      region: "us-east-1",
    });

    new EC2.Instance(this, "Hello", {
      ami: "ami-2757f631",
      instanceType: "t2.micro",
      tags: {
        myConfig: this.node.getContext("myConfig"),
      },
    });
  }
}

const app = new App({ context: { myConfig: "value" } });
new MyStack(app, "hello-cdktf");
app.synth();
```

## Convert an HCL Project to a CDKTF TypeScript Project

You can initialize a new CDKTF TypeScript project from an existing project written in HashiCorp Configuration Language (HCL). This option is currently limited to the `typescript` template.

To convert an existing HCL project, add `--from-terraform-project` to the `init` command with the TypeScript template.

```
$ cdktf init --template=typescript --from-terraform-project /path/to/my/tf-hcl-project
```

### Usage Example

Consider the HCL configuration below.

```hcl
# File: /tmp/demo/main.tf

terraform {
  required_providers {
    random = {
      source = "hashicorp/random"
      version = "3.1.0"
    }
  }
}

provider "random" {
}

resource "random_pet" "server" {
}
```

Run the command to convert the HCL project in a new folder.

```sh
cdktf init --template=typescript --from-terraform-project /tmp/demo --local
```

CDKTF bootstraps a Typescript project and generates the equivalent configuration in JSON.

```jsonc
// File: /tmp/cdktf-demo/cdktf.json

{
  "language": "typescript",
  "app": "npm run --silent compile && node main.js",
  "projectId": "83684893-0e58-4a71-989a-ecb7c593a690",
  "terraformProviders": ["hashicorp/random@3.1.0"],
  "terraformModules": [],
  "context": {
    "excludeStackIdFromLogicalIds": "true",
    "allowSepCharsInLogicalIds": "true"
  }
}
```

```ts
// File: /tmp/cdktf-demo/main.ts

/*Provider bindings are generated by running cdktf get.
See https://github.com/hashicorp/terraform-cdk/blob/main/docs/working-with-cdk-for-terraform/using-providers.md#importing-providers-and-modules for more details.*/
import * as random from "./.gen/providers/random";

import { Construct } from "constructs";
import { App, TerraformStack } from "cdktf";

class MyStack extends TerraformStack {
  constructor(scope: Construct, name: string) {
    super(scope, name);

    new random.RandomProvider(this, "random", {});
    new random.Pet(this, "server", {});
  }
}

const app = new App();
new MyStack(app, "cdktf-demo");
app.synth();
```

## Convert HCL Files to CDKTF Format

Use the `cdktf convert` command to convert individual HCL files to CDKTF-compatible files in your preferred programming language. Refer to the [`cdktf convert` command documentation](/docs/cdktf/cli-reference/commands.html) for more information.