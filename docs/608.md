# 可重复使用的 YAML 与 CircleCI 球体-章鱼部署

> 原文：<https://octopus.com/blog/reusable-yaml-with-circleci-orbs>

[![Reusable YAML with CircleCI orbs](img/ce1c537d64425c266c3f31b257d42072.png)](#)

持续集成和持续交付平台中的一个增长趋势是提供将管道定义为代码的能力，通常使用 YAML。这个领域的领导者之一是 CircleCI。在这篇文章中，我们将看看 CircleCI 配置，包括如何使用和创作 CircleCI `orbs`。orb 是一个可重用的 YAML 块，可以跨 CircleCI 管道使用。您可以从 orb 中引用功能，而不是在多个文件之间复制和粘贴相同的代码，并保持管道畅通。

## 什么是 CircleCI

CircleCI 是一个持续集成平台，它使用基于 YAML 的配置在 Docker 容器或虚拟机上执行作业。这些工作可以被编织到一个或多个工作流中，这些工作流将在您将代码提交到源代码控制时执行。

下面是一个名为`build`的 CircleCI 作业示例:

```
jobs:
  build:
    docker:
      - image: mcr.microsoft.com/dotnet/core/sdk:2.1.607-stretch
    environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >>
    steps:
      - checkout
      - run:
          name: Install Cake
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet tool install -g Cake.Tool --version 0.35.0
      - run:
          name: Build
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet cake build.cake --target="Publish" --packageVersion="$PACKAGE_VERSION"
      - persist_to_workspace:
          root: publish
          paths:
            - "*" 
```

让我们一点一点地把它分解。

首先，我们定义在哪个 docker 映像上执行我们的工作:

```
 docker:
      - image: mcr.microsoft.com/dotnet/core/sdk:2.1.607-stretch 
```

我们为包含 CircleCI 管道号的包版本定义了一个环境变量:

```
 environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >> 
```

我们定义我们的步骤。第一步是检查我们的源代码:

```
 steps:
      - checkout 
```

第二步在执行作业的容器上安装 Cake:

```
 - run:
          name: Install Cake
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet tool install -g Cake.Tool --version 0.35.0 
```

第三步调用 cake 来构建我们的项目:

```
 - run:
          name: Build
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet cake build.cake --target="Publish" --packageVersion="$PACKAGE_VERSION" 
```

最后一步是将我们在`publish`文件夹中发布的应用程序保存到 CircleCI 工作区。工作区用于跨多个作业共享资产:

```
 - persist_to_workspace:
          root: publish
          paths:
            - "*" 
```

让我们来看一个使用该作业的工作流。在定义工作流的作业时，我们可以将一个作业设置为依赖于一个或多个其他作业。我们已经创建了一个名为`build`的工作流，它将依次运行作业`build`、`package`、`push`、`create-release`和`deploy-release`:

```
workflows:
  version: 2
  build:
    jobs:
      - build
      - package:
          requires:
            - build
      - push:
          requires:
            - package
      - create-release:
          requires:
            - push
      - deploy-release:
          requires:
            - create-release 
```

## 可重复使用的 YAML 和 CircleCI 球体

比方说，我们有多个使用 Cake 构建的项目，步骤都差不多。对于每个项目来说，使用一个标准的过程而不在每个配置中定义相同的 YAML 不是很好吗？

这就是球体出现的地方！orb 定义了可重用的执行器(您的作业将在其中运行的环境)、可在作业中使用的命令以及可在工作流中使用的作业。

### 使用球体

要使用 orb，需要在配置中导入一个`orbs`部分。在下一节中，我们添加了松弛球，命名为`slack`，并将其映射到球`circleci/slack@3.4.2`。`circleci`是包含 orb 的名称空间。`slack`是宝珠的名字。`3.4.2`是我们要用的 orb 的版本。

在我们的`build`工作中，我们添加了一个新步骤`- slack\notify`。这一步是来自那个球的`notify`命令。您可以查看命令的[来源。这是相当多的 YAML，我们不需要包括在我们的配置中。](https://circleci.com/orbs/registry/orb/circleci/slack#commands-notify)

```
version: 2.1

orbs:
  slack: circleci/slack@3.4.2

jobs:
  build:
    docker:
      - image: mcr.microsoft.com/dotnet/core/sdk:2.1.607-stretch
    environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >>
    steps:
      - checkout
      - run:
          name: Install Cake
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet tool install -g Cake.Tool --version 0.35.0
      - run:
          name: Build
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet cake build.cake --target="Publish" --packageVersion="$PACKAGE_VERSION"
      - persist_to_workspace:
          root: publish
          paths:
            - OctopusSamples.OctoPetShop.Database
            - OctopusSamples.OctoPetShop.Infrastructure
            - OctopusSamples.OctoPetShop.ProductService
            - OctopusSamples.OctoPetShop.ShoppingCartService
            - OctopusSamples.OctoPetShop.Web
      - slack/notify:
          message: Build ${PACKAGE_VERSION} succeeded 
```

因此，让我们为我们的蛋糕步骤创建一个球体。

### 创建内联球体

在上面的例子中，我们在配置中导入了一个 orb。我们也可以在配置中直接定义一个 orb:

```
orbs:
  cake:
    executors:
      cake-executor:
        docker:
          - image: mcr.microsoft.com/dotnet/core/sdk:2.1.607-stretch
    jobs:
      build:
        executor: cake-executor
        parameters:
          cake_version:
            default: "0.35.0"
            type: string
          package_version:
            default: "$PACKAGE_VERSION"
            type: string
          publish_path:
            default: "publish"
            type: string
          target:
            default: "Publish"
            type: string
        steps:
          - checkout
          - run:
              name: Install Cake
              command: |
                export PATH="$PATH:$HOME/.dotnet/tools"
                dotnet tool install -g Cake.Tool --version << parameters.cake_version >>
          - run:
              name: Build
              command: |
                export PATH="$PATH:$HOME/.dotnet/tools"
                dotnet cake build.cake --target="<< parameters.target >>" --packageVersion="<< parameters.package_version >>"
          - persist_to_workspace:
              root: << parameters.publish_path >>
              paths:
                - "*" 
```

在上面的 orb 中，我们创建了一个执行器，它定义了要使用的 Docker 映像。我们还定义了一个`build`作业，它接受 Cake 版本、包版本、发布路径和 Cake 目标作为参数。然后，它以类似于我们之前的步骤使用这些参数。

定义 orb 后，我们可以将工作流更新为以下内容:

```
workflows:
  version: 2
  build:
    jobs:
      - cake/build:
          package_version: 1.3.<< pipeline.number >>
      - package:
          requires:
            - cake/build
      - push:
          requires:
            - package
      - create-release:
          requires:
            - push
      - deploy-release:
          requires:
            - create-release 
```

现在我们的工作流将使用我们在配置中定义的蛋糕 orb 中的`build`作业。

### 发布球体

为了在多个项目中使用这个 orb，我们需要发布它。为此，我们需要安装和配置 [CircleCI CLI](https://circleci.com/docs/2.0/orb-author-cli/#install-the-cli-for-the-first-time) 。

完成后，我们为我们的组织创建一个名称空间:

```
circleci namespace create octopus-samples github OctopusSamples # replace with your values 
```

然后，我们在名称空间中创建 orb:

```
circleci orb create octopus-samples/cake # replace with your values 
```

现在要将内嵌的 orb 发布到 CircleCI。我们需要将 orb 定义从我们的配置中移到它自己的 YAML 文件中。我们就把它命名为 orb.yml。

```
version: 2.1

description: |
  Orb for using Cake with OctopusSamples.

executors:
  cake-executor:
    docker:
      - image: mcr.microsoft.com/dotnet/core/sdk:2.1.607-stretch
jobs:
  build:
    executor: cake-executor
    parameters:
      cake_version:
        default: "0.35.0"
        type: string
      package_version:
        default: "$PACKAGE_VERSION"
        type: string
      publish_path:
        default: "publish"
        type: string
      target:
        default: "Publish"
        type: string
    steps:
      - checkout
      - run:
          name: Install Cake
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet tool install -g Cake.Tool --version << parameters.cake_version >>
      - run:
          name: Build
          command: |
            export PATH="$PATH:$HOME/.dotnet/tools"
            dotnet cake build.cake --target="<< parameters.target >>" --packageVersion="<< parameters.package_version >>"
      - persist_to_workspace:
          root: << parameters.publish_path >>
          paths:
            - "*" 
```

然后我们可以发布它:

```
circleci orb publish orb.yml octopus-samples/cake@0.0.1 
```

转眼间。我们有一个[发布的 orb](https://circleci.com/orbs/registry/orb/octopus-samples/cake) 和我们的蛋糕制作工作。现在我们可以更新项目的配置，用发布的 orb 替换内联 orb:

```
orbs:
  cake: octopus-samples/cake@0.0.1 
```

## 实验章鱼 CLI orb

说到已发布的球体，如果你同时使用 CircleCI 和 Octopus，看看我们的[实验 Octopus CLI 球体](https://circleci.com/orbs/registry/orb/octopusdeploylabs/octopus-cli)。有了这个 orb，您可以使用 Octopus CLI 中的命令子集来管理您的包、发布和部署:

```
orbs:
  octo: octopusdeploylabs/octopus-cli@0.0.2

jobs:
  package:
    docker:
      - image: ubuntu:18.04
    environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >>
    steps:
      - octo/install-tools
      - attach_workspace:
          at: publish
      - octo/pack:
          id: "OctopusSamples.OctoPetShop.Database"
          version: "$PACKAGE_VERSION"
          base_path: "publish/OctopusSamples.OctoPetShop.Database"
          out_folder: "package"
      - persist_to_workspace:
          root: package
          paths:
            - OctopusSamples*
  push:
    docker:
      - image: ubuntu:18.04
    environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >>
    steps:
      - octo/install-tools
      - attach_workspace:
          at: package
      - octo/push:
          package: "package/OctopusSamples.OctoPetShop.Database.${PACKAGE_VERSION}.zip"
          server: "$OCTOPUS_SERVER"
          api_key: "$OCTOPUS_API_KEY"
          debug: true
      - octo/build-information:
          package_id: "OctopusSamples.OctoPetShop.Database"
          version: "$PACKAGE_VERSION"
          server: "$OCTOPUS_SERVER"
          api_key: "$OCTOPUS_API_KEY"
          debug: true
  create-release:
    docker:
      - image: ubuntu:18.04
    environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >>
    steps:
      - octo/install-tools
      - octo/create-release:
          project: "Octo Pet Shop"
          server: "$OCTOPUS_SERVER"
          api_key: "$OCTOPUS_API_KEY"
          release_number: $PACKAGE_VERSION 
```

另外，我们可以继续使用内联 orb 来进一步减少重复。让我们围绕`octo-exp\pack`做一个包装:

```
orbs:
  cake: octopus-samples/cake@0.0.1
  octo: octopusdeploylabs/octopus-cli@0.0.2
  octopetshop:
    orbs:
      octo: octopusdeploylabs/octopus-cli@0.0.2
    commands:
      pack:
        parameters:
          id:
            type: string
        steps:
          - octo/pack:
              id: "<< parameters.id >>"
              version: $PACKAGE_VERSION
              base_path: "publish/<< parameters.id >>"
              out_folder: "package"

jobs:
  package:
    executor:
      - name: octo/default
    environment:
      PACKAGE_VERSION: 1.3.<< pipeline.number >>
    steps:
      - octo/install-tools
      - attach_workspace:
          at: publish
      - octopetshop/pack:
          id: "OctopusSamples.OctoPetShop.Database"
      - octopetshop/pack:
          id: "OctopusSamples.OctoPetShop.Infrastructure"
      - octopetshop/pack:
          id: "OctopusSamples.OctoPetShop.ProductService"
      - octopetshop/pack:
          id: "OctopusSamples.OctoPetShop.ShoppingCartService"
      - octopetshop/pack:
          id: "OctopusSamples.OctoPetShop.Web"
      - persist_to_workspace:
          root: package
          paths:
            - OctopusSamples* 
```

因为我们对 octo/pack 的调用将遵循相同的格式，但对`id`的值不同，所以我们可以使用`octopetshop\pack`来抽象这个逻辑。

## 结论

CircleCI Orbs 是为基于 YAML 的管道创建可重用命令或作业的一种方式。您可以将 orb 发布到 CircleCI，以便跨项目或与其他组织共享。您还可以使用内联 orb 来帮助开发或者减少您自己的配置中的干扰。

查看 [CircleCI Orbs](https://circleci.com/orbs/) 了解更多信息，查看[实验章鱼 CLI orb](https://circleci.com/orbs/registry/orb/octopusdeploylabs/octopus-cli) 了解如何一起使用 CircleCI 和章鱼的详细信息。