# 比特斗管道-Octo.exe 集装箱 Redux -八达通部署

> 原文：<https://octopus.com/blog/bitbucket-pipelines-redux>

[![Bitbucket pipelinse and Octopus Deploy](img/77156b2801aabc37776b329f459ef826.png)](#)

自从这篇文章首次发表以来，我们已经重新命名了 Octo.exe，现在它是 Octopus CLI，更多信息，请参见这篇文章: [Octopus release 2020.1](https://www.octopus.com/blog/octopus-release-2020-1) 。

早在二月份(几年前在互联网领域)，我们[发表了一篇关于如何通过 Octopus Deploy 将你的](https://octopus.com/blog/continuous-delivery-bitbucket-pipelines) [Bitbucket 管道](https://bitbucket.org/product/features/pipelines)构建过程与部署联系起来的文章。自从我们写这篇文章以来，我们已经开始发布我们的[octo.exe](https://octopus.com/docs/octopus-rest-api/octopus-cli)命令行工具的最新容器映像，当在 Octopus 门户之外编写脚本时，它将加速您的持续部署过程，特别是对于这些基于容器的构建链。

[![The Original Pipeline](img/11b8c787b2d0c48b0dbe963d49aca6f6.png "float=right")](#)

对于那些错过了前一篇文章的人来说，Bitbucket Pipelines 提供了一种非常简单且低成本的方法，可以从 Bitbucket 代码仓库自动构建。将它与 Octopus Deploy 结合起来意味着您可以正确地管理您的部署，同时仍然减少从代码到应用程序的开销。它使用容器来执行每一步，这是构建工具的一种令人敬畏的新方法。许多供应商已经开始使用这种策略，这意味着每个构建过程都可以精确地适应所讨论的应用程序的需求。多个不同的平台和工具版本可以按需在同一个构建基础设施上并行运行，而不会在构建之间产生任何冲突或污染。

octo.exe 容器现在[发布到 DockerHub](https://hub.docker.com/r/octopusdeploy/octo/) 上，同时提供了 alpine 和 nanoserver 基础映像。这意味着您可以在类似于 BitBucket 提供的容器化构建链中使用 octo.exe。扩展 Andy 在上一篇文章中概述的思想，我们现在可以避免安装任何压缩工具或执行任何复杂的 curl 请求。

节点根中的`bitbucket-pipelines.yml`。JS 项目可以简单到

```
image: node:6.9.4

pipelines:
  default:
    - step:
        name: Build And Test
        caches:
          - node
        script:
          - npm install
          - npm test
          - npm prune --production
        artifacts:
          - "*/**"
    - step:
        name: Deploy to Octopus
        image: octopusdeploy/octo:4.37.0-alpine
        script:
          - export VERSION=1.0.$BITBUCKET_BUILD_NUMBER
          - octo pack --id $BITBUCKET_REPO_SLUG --version $VERSION --outFolder ./out --format zip
          - octo push --package ./out/$BITBUCKET_REPO_SLUG.$VERSION.zip  --server $OCTOPUS_SERVER --apiKey $OCTOPUS_APIKEY 
```

在我们构建并测试了我们的节点应用程序之后(这一步可以通过提取测试结果等来改进，但是现在让我们忽略它)，我们指示管道将工作目录中的所有文件作为工件传递。这些文件将出现在后续步骤的工作目录中。注意，名为`Deploy to Octopus`的第二步使用了`octopusdeploy/octo:4.37.0-alpine`容器。这里基本上只需要运行两个命令；`octo pack`和`octo push`。我们可以使用 Bitbucket 提供的环境变量以存储库命名包，并根据构建号生成一个版本，然后使用自定义的[安全变量](https://confluence.atlassian.com/bitbucket/environment-variables-794502608.html)通过项目设置来设置服务器 URL 和 apiKey。

与其重复 Andy 在[上一篇文章](https://octopus.com/blog/continuous-delivery-bitbucket-pipelines)中关于 Bitbucket 如何工作的内容，我建议您阅读他的文章，了解更多关于建立 Bitbucket 管道和 Octopus 端到端部署 CI 流程的详细信息。

上面的配置变化显示了如何比以往更容易地将 Octopus Deploy 集成到您的构建系统中，并且该方法将在大多数容器化的构建系统中工作，从 Bitbucket 管道到 [CircleCI](https://circleci.com) 。立即尝试，简化您的部署，现在容器数量增加了一倍！

## 了解更多信息