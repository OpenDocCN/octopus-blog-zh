# Bitbucket 的 Octopus 管道:octopus-cli-run - Octopus Deploy

> 原文：<https://octopus.com/blog/octopus-bitbucket-pipe>

[![Bitbucket Pipelines](img/394c6a78b8f08d81fb0813a53b4cd66d.png)](#)

在之前的一篇文章中，我写了[如何创建一个 Bitbucket 管道并将其与 Octopus Deploy](/blog/bitbucket-pipes-and-octopus-deploy) 集成。如果您是第一次开始使用管道，值得一读。

在本帖中，我将向您概述 Octopus - [octopus-cli-run](https://bitbucket.org/octopusdeploy/octopus-cli-run/) 的新实验性 Bitbucket 管道。如果你有兴趣尝试实验管道，你可以用它来运行来自 [Octopus CLI](https://octopus.com/docs/octopus-rest-api/octopus-cli/) 的命令，允许你进一步将你的 Atlassian[bit bucket Pipeline](https://bitbucket.org/product/features/pipelines)与 Octopus 集成来管理你的包、发布和部署。

## 在这篇文章中

## 管道 YAML 定义

管道的基本定义包括对托管在[位桶](https://bitbucket.org/octopusdeploy/octopus-cli-run/)上的仓库的引用。它还在 Docker Hub 上被发布为[octopus/octopus-CLI-run](https://hub.docker.com/r/octopipes/octopus-cli-run/)。

它有一个必需的`CLI_COMMAND`变量。这是要运行的 CLI 命令。

要在您的`bitbucket-pipelines.yml`文件中使用管道，请将下面的 YAML 代码片段添加到脚本部分:

```
- pipe: octopusdeploy/octopus-cli-run:0.13.0
  variables:
    CLI_COMMAND: "<string>"
    # EXTRA_ARGS: ['<string>','<string>' ..] # Optional
    # DEBUG: "<boolean>" # Optional 
```

管道还提供了一个名为`EXTRA_ARGS`的*可选*数组变量，您可以使用它来包含指定命令的任何附加命令行参数。

### 管道变量定义

位桶管道和管道中的变量被配置为[环境变量](https://confluence.atlassian.com/bitbucket/variables-in-pipelines-794502608.html)。由于`octopus-cli-run`管道包含许多命令，所需的具体变量取决于您正在使用的命令。参见[自述文件](https://bitbucket.org/octopusdeploy/octopus-cli-run/src/master/README.md#markdown-header-variables)了解每个命令所需变量的详细信息。

## 支持的命令

这个`octopus-cli-run`管道是用最常用的 CLI 命令编写的，它实际上是建立在 [Octopus CLI Docker 映像](https://hub.docker.com/r/octopusdeploy/octo/)之上的。这包括以下能力:

*   使用 [pack](https://octopus.com/docs/octopus-rest-api/octopus-cli/pack) 打包您的文件或构建工件。
*   使用 [push](https://octopus.com/docs/octopus-rest-api/octopus-cli/pack) 将包发送到 Octopus 内置存储库。
*   使用[构建信息](https://octopus.com/docs/octopus-rest-api/octopus-cli/build-information)将构建信息推送到 Octopus。
*   使用 [create-release](https://octopus.com/docs/octopus-rest-api/octopus-cli/create-release) 自动创建发布。
*   使用[部署-发布](https://octopus.com/docs/octopus-rest-api/octopus-cli/create-release)部署已经创建的发布。

接下来，我们将使用 [Bitbucket](https://bitbucket.org/octopussamples/petclinic/) 上提供的 **PetClinic** 示例应用程序来探索每个命令的管道步骤。为了保持简单，这些步骤已经减少到所需的最小定义。

### 包装

`pack`命令允许你从磁盘上的文件创建[包](https://octopus.com/docs/packaging-applications)(zip 或 nupkg 格式)，而不需要`.nuspec`或`.csproj`文件。

要创建包，请定义如下步骤:

```
- step:
    name: octo pack mysql-flyway
    script:
      - pipe: octopusdeploy/octopus-cli-run:0.13.0
        variables:
          CLI_COMMAND: 'pack'
          ID: 'petclinic.mysql.flyway'
          FORMAT: 'Zip'
          VERSION: '1.0.0.0'
          SOURCE_PATH: 'flyway'
          OUTPUT_PATH: 'flyway'
    artifacts:
      - "flyway/*.zip" 
```

这会打包`flyway`文件夹，并在同一个文件夹中创建一个名为`petclinic.mysql.flyway.1.0.0.0.zip`的 zip 文件。

### 推

`push`命令使您能够推送包(。zip，。nupkg，。战争等)到章鱼[的内置储存库](https://octopus.com/docs/packaging-applications/package-repositories/built-in-repository)

还支持同时推送多个包。要执行多包`push`，定义如下步骤:

```
- step:
    name: octo push
    script:
      - pipe: octopusdeploy/octopus-cli-run:0.13.0
        variables:
          CLI_COMMAND: 'push'
          OCTOPUS_SERVER: $OCTOPUS_SERVER
          OCTOPUS_APIKEY: $OCTOPUS_API_KEY
          OCTOPUS_SPACE: $OCTOPUS_SPACE
          PACKAGES: [ "./flyway/petclinic.mysql.flyway.1.0.0.0.zip", "target/petclinic.web.1.0.0.0.war" ] 
```

这将把`petclinic.mysql.flyway.1.0.0.0.zip`和`petclinic.web.1.0.0.0.war`包推送给 Octopus。

### 构建信息

`build-information`命令帮助您将关于您的构建的信息(编号、URL、提交)传递给 Octopus。这些信息可以在 Octopus 中查看，也可以在[发行说明](https://octopus.com/docs/managing-releases/release-notes)和[部署说明](https://octopus.com/docs/managing-releases/deployment-notes)中使用。

如果您已经创建了一个构建信息文件，那么您可以使用`FILE`变量将它提供给命令。如果没有提供变量，管道将生成自己的构建信息文件，并将其发送给 Octopus。

要推送自动生成的构建信息文件，请定义如下步骤:

```
- step:
    name: octo build-information
    script:
      - pipe: octopusdeploy/octopus-cli-run:0.13.0
        variables:
          CLI_COMMAND: 'build-information'
          OCTOPUS_SERVER: $OCTOPUS_SERVER
          OCTOPUS_APIKEY: $OCTOPUS_API_KEY
          OCTOPUS_SPACE: $OCTOPUS_SPACE
          VERSION: '1.0.0.0'
          PACKAGE_IDS: ['petclinic.web'] 
```

这将创建构建信息，将它与`petclinic.web`包的版本`1.0.0.0`相关联，并将其推送到 Octopus。

### 创建发布

`create-release`命令允许您在 Octopus 中创建一个版本。您使用`PROJECT`变量指定项目来创建发布。

或者，您也可以将该版本部署到一个或多个环境中。为了实现这一点，您应该使用全局`EXTRA_ARGS`数组变量并提供适当的选项。例如:

`EXTRA_ARGS: ['--deployTo', 'Development', '--guidedFailure', 'True']`

要创建一个版本，并让 Octopus 选择要使用的版本，创建如下步骤:

```
- step:
    name: octo create-release
    script:
      - pipe: octopusdeploy/octopus-cli-run:0.13.0
        variables:
          CLI_COMMAND: 'create-release'
          OCTOPUS_SERVER: $OCTOPUS_SERVER
          OCTOPUS_APIKEY: $OCTOPUS_API_KEY
          OCTOPUS_SPACE: $OCTOPUS_SPACE
          PROJECT: $OCTOPUS_PROJECT 
```

### 部署发布

`deploy-release`命令允许您部署已经创建的版本。使用`PROJECT`和`RELEASE_NUMBER`变量来指定项目和发布版本号以部署发布版本。

通过使用名称或 ID 在`DEPLOY_TO`变量中指定要部署到的环境，如下所示:

`DEPLOY_TO: ['Environments-1', 'Development', 'Staging', 'Test']`

要为一个项目将`latest`版本部署到`Development`，创建如下步骤:

```
- step:
    name: octo deploy-release
    script:
      - pipe: octopusdeploy/octopus-cli-run:0.13.0
        variables:
          CLI_COMMAND: 'deploy-release'
          OCTOPUS_SERVER: $OCTOPUS_SERVER
          OCTOPUS_APIKEY: $OCTOPUS_API_KEY
          OCTOPUS_SPACE: $OCTOPUS_SPACE
          PROJECT: $OCTOPUS_PROJECT
          RELEASE_NUMBER: 'latest'
          DEPLOY_TO: ['Development'] 
```

## 使用管道

最后，让我们看看如何在多个步骤中使用管道来组成完整的位桶管道:

```
image: maven:3.6.1

pipelines:
  branches:
    master:
      - step:
          name: build petclinic
          caches:
            - maven
          script:
            - mvn -B verify -DskipTests -Dproject.versionNumber=1.0.0.0 -DdatabaseUserName=$DatabaseUserName -DdatabaseUserPassword=$DatabaseUserPassword -DdatabaseServerName=$DatabaseServerName -DdatabaseName=$DatabaseName
          artifacts:
          - "target/*.war"
      - step:
          name: octo pack mysql-flyway
          script:
            - pipe: octopusdeploy/octopus-cli-run:0.13.0
              variables:
                CLI_COMMAND: 'pack'
                ID: 'petclinic.mysql.flyway'
                FORMAT: 'Zip'
                VERSION: '1.0.0.0'
                SOURCE_PATH: 'flyway'
                OUTPUT_PATH: './flyway'
          artifacts:
            - "flyway/*.zip"
      - step:
          name: octo push
          script:
            - pipe: octopusdeploy/octopus-cli-run:0.13.0
              variables:
                CLI_COMMAND: 'push'
                OCTOPUS_SERVER: $OCTOPUS_SERVER
                OCTOPUS_APIKEY: $OCTOPUS_API_KEY
                OCTOPUS_SPACE: $OCTOPUS_SPACE
                PACKAGES: [ "./flyway/petclinic.mysql.flyway.1.0.0.0.zip", "target/petclinic.web.1.0.0.0.war" ]
      - step:
          name: octo build-information
          script:
            - pipe: octopusdeploy/octopus-cli-run:0.13.0
              variables:
                CLI_COMMAND: 'build-information'
                OCTOPUS_SERVER: $OCTOPUS_SERVER
                OCTOPUS_APIKEY: $OCTOPUS_API_KEY
                OCTOPUS_SPACE: $OCTOPUS_SPACE
                VERSION: '1.0.0.0'
                PACKAGE_IDS: ['petclinic.web']
      - step:
          name: octo create-release
          script:
            - pipe: octopusdeploy/octopus-cli-run:0.13.0
              variables:
                CLI_COMMAND: 'create-release'
                OCTOPUS_SERVER: $OCTOPUS_SERVER
                OCTOPUS_APIKEY: $OCTOPUS_API_KEY
                OCTOPUS_SPACE: $OCTOPUS_SPACE
                PROJECT: $OCTOPUS_PROJECT
      - step:
          name: octo deploy-release
          script:
            - pipe: octopusdeploy/octopus-cli-run:0.13.0
              variables:
                CLI_COMMAND: 'deploy-release'
                OCTOPUS_SERVER: $OCTOPUS_SERVER
                OCTOPUS_APIKEY: $OCTOPUS_API_KEY
                OCTOPUS_SPACE: $OCTOPUS_SPACE
                PROJECT: $OCTOPUS_PROJECT
                RELEASE_NUMBER: 'latest'
                DEPLOY_TO: ['Development'] 
```

就是这样！

您可以在[位桶](https://bitbucket.org/octopussamples/petclinic/src/master/bitbucket-pipelines.yml)上查看完整的 PetClinic `bitbucket-pipelines.yml`文件。

**样本 Octopus 项目**
您可以在我们的[样本](https://g.octopushq.com/TargetWildflySamplePetClinic)实例中看到 PetClinic Octopus 项目。

## 结论

使用位桶管道确实有助于简化位桶管道中的配置，并且作为作者有助于促进您的操作的重用。查看 [Bitbucket Pipelines](https://bitbucket.org/product/features/pipelines/integrations) 了解更多信息，查看[实验性 Octopus Pipe](https://bitbucket.org/octopusdeploy/octopus-cli-run/src/master/README.md) 了解如何一起使用 Bitbucket 和 Octopus 的更多细节。