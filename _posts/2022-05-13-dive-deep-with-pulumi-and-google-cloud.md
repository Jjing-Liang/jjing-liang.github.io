---
title:  "Dive deeper with Pulumi and Google Cloud"
layout: post
---

本文主要是在官网的一篇[Google Cloud Run: Serverless Containers](https://www.pulumi.com/blog/google-cloud-run-serverless-containers/)所提供的例子上进行自定义配置的一次实践，如果你遇到类似问题，希望这篇文章可以帮助到你。

- 自行管理Pulumi metadata，不使用默认的Pulumi service
- 使用Google KMS加密secret，不使用默认的Pulumi service
- 利用`Pulumi Stack`快速部署多个环境


## Pulumi简介

直接引用[官方介绍](https://www.pulumi.com/docs/intro/concepts/)：

> Pulumi is a modern infrastructure as code platform. It leverages existing programming languages—TypeScript, JavaScript, Python, Go, .NET, Java, and markup languages like YAML—and their native ecosystem to interact with cloud resources through the Pulumi SDK. A downloadable CLI, runtime, libraries, and a hosted service work together to deliver a robust way of provisioning, updating, and managing cloud infrastructure.
> 

如果对于 [Infrastructure as Code](https://www.pulumi.com/what-is/what-is-infrastructure-as-code/) 不太熟悉，可以先去了解看看。

Pulumi特性较多，2022年5月刚刚宣布支持Java，这里列出部分特性：

- 支持70+ cloud provider，详见[Pulumi registry](https://www.pulumi.com/registry/)
- Team可以选择语言编写架构代码，更好的利用自己所熟悉语言的优势：Python、Go、JavaScript、TypeScript、C#,、Java、YAML，2022年5月刚刚宣布支持Java
- 更好与软件工程实践结合：modularity, testing, and CI/CD

官方也提供了与[常见的IaS工具](https://www.pulumi.com/docs/intro/vs/)（例如[Terraform](https://www.pulumi.com/docs/intro/vs/terraform/)）详细对比。

![Pulumi evolution](https://www.pulumi.com/blog/pulumi-universal-iac/pulumi-evolution.png)


## Using a self-managed backend store metadata

在[How Pulumi works](https://www.pulumi.com/docs/intro/concepts/how-pulumi-works/)中提到Pulumi会对比期望状态与当前状态，以此决定需要创建、更新和删除哪些资源。这些状态数据就是metadata，Pulumi默认会将metadata存储在Pulumi的后端-Pulumi service中，不过也提供了自行管理metadata，存储在用户信任的后端中，例如AWS S3, Azure Blob Store, Google Cloud Storage，本地存储等。为了探索Pulumi在GCP的应用，所以选择将数据存储在Google Cloud Storage，不存储在Pulumi service中。

如果你的架构还没有开始使用pulumi，可以在最开始的时候进行self-managed backend的配置。如果你已经使用了默认的Pulumi service或者其他backend，Pulumi也支持[backend的迁移](https://www.pulumi.com/docs/intro/concepts/state/#migrating-between-backends)。

### Step 1: 在Google Cloud Storage（GCS）创建bucket及folder

![GCS](/assets/img/gcs.png)

### Step 2: 运行`pulumi login`

```bash
# 使用bucket path或者folder path均可
pulumi login gs://<my-pulumi-bucket-or-folder>
```

通过上面两步配置，metadata就会存储在GCS中，此时如果运行`pulumi up`，GCS中就会出现`.pulumi/`的folder，如下图所示。`.pulumi`主要包含三个文件夹：backups，history，stacks。

![GCS folder](/assets/img/gcs-pulumi.png)

```bash

.pulumi   
└───backups
│   └───subfolder
│       │   ...
└───history
│   └───subfolder
│       │   ...
└───state
│   └───subfolder
│       │   ...
```

## Secret

在上面提及的metadata中记录了所有的状态，状态中不可避免会保存一些敏感数据，例如数据库密码或者service token。Pulumi支持对secret进行加密保护，从而在状态数据中避免以明文出现。与metadata类似，默认使用Pulumi service提供，一个stack对应一个加密密钥，同时也支持指定secret provider。

这里简单说明一下`stack`。你可能在[Get started with Pulumi](https://www.pulumi.com/docs/get-started/gcp/create-project/)中注意到可以指定“stack name”，在目录中也会创建出一个与stack name相对应的配置文件，例如`Pulumi.dev.yaml`。Pulumi对stack的描述是：

> A stack is an isolated, independently configurable instance of a Pulumi program.
> 

Stack常见会按照开发阶段分类（例如dev，staging，prod等）或者特性分支分类（例如feature-x等），关于stack的具体细节可以参见[Pulumi Stacks](https://www.pulumi.com/docs/intro/concepts/stack/)。

![Pulumi get started terminal screenshot](/assets/img/console-pulumi-new-stack.png)

针对不同stack所配置的环境变量或者secret会以key-value的形式存储在`Pulumi.<stack-name>.yaml`。

同样基于探索的目的，这里使用Google cloud Key Management Service (KMS)作为secret provider。

### Step 1: GCP Key Management创建密钥环（key ring）和密钥

这里使用GCP的Key management的对称密钥，创建对应的密钥环和密钥即可，其他配置可以参考[GCP key management文档](https://cloud.google.com/kms/docs/creating-keys)。需要注意，GCP目前不支持重命名或者删除密钥环及密钥。

![gcp create kms key](/assets/img/gcp-kms-create-key.png)

### Step 2: 指定secret provider

Secret provider需要在初始化stack时一并设置。

```bash
pulumi stack init <stack-name> --secrets-provider="gcpkms://projects/<gcp-project-id>/locations/<key-location>/keyRings/<key-ring-name>/cryptoKeys/<key-name>"

# 以stack name为qa为例
pulumi stack init qa --secrets-provider="gcpkms://projects/my-gcp-project/locations/global/keyRings/pulumi-kms-ring-name/cryptoKeys/pulumi-kms-key-name"
```

初始化后会在Pulumi.qa.yaml中自动生成两行配置(encryptedKey为非真实key，仅用于举例)

![pulumi generated key](/assets/img/vscode-kms-provider.jpg)

### Step 3: 设置secret

```bash
pulumi config set --secret db-password mockPwd
```

![pulumi secret](/assets/img/vscode-kms-secret.jpg)

## Multiple environment

如果一个服务存在多个环境，上文也提到可以用`stack`进行划分和部署，所有在pulumi中定义和变更的基础架构资源都记录在对应的`stack instance`中。这种划分模式适用于多个环境之间差异性较小的情况，使用同一份Infrastructure的代码可以快速部署多个环境，可复用性较强。Kief Morri在其所写的**[Infrastructure as Code, 2nd Edition](https://www.oreilly.com/library/view/infrastructure-as-code/9781098114664/)**中也有提及到。

![IaC structring stacks](/assets/img/book-iac-env.png)

Pulumi提供了全面的command line interface（CLI），以下是在stack中较为常用的命令：

```bash

# 初始化stack
pulumi stack init my-app-production

# 罗列stacks
pulumi stack ls

# 查看当前stack的详细信息
pulumi stack

# 选择slack
pulumi stack select <stack-name>

# 导出stack数据到本地
pulumi stack export --show-secrets --file my-app-production.checkpoint.json

# 导入指定stack数据到当前stack
pulumi stack import --file my-app-production.checkpoint.json
```

## More with Pulumi

基于想探索如何更好的实践IaC的理念，除上述内容之外，还尝试了pulumi其他特性：

- 与CI/CD集成：CircleCI
- 导入已有云平台资源
- 对不同的服务选用不同的语言：Typescript，Java

这里推荐感兴趣的朋友可以直接参考Pulumi[官方文档](https://www.pulumi.com/docs/)和[Github](https://github.com/pulumi/pulumi)即可，它们基本解答了我在实践中遇到的各种问题，也可以帮助了解Pulumi背后的思想。