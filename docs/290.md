# 我们最喜欢的 10 个 GitHub 动作——Octopus Deploy

> 原文：<https://octopus.com/blog/githubactions-ten-favorite-actions>

尽管对持续集成(CI)世界来说相对较新，GitHub 的“动作”的加入已经见证了其强大的社区构建了直接插入到您的存储库中的有用任务。

操作允许您运行非标准任务，以帮助您测试、构建和将您的工作推送到您的部署工具。

排名不分先后，以下是我们的 10 个最爱，外加如何安装它们的。

## 1:测试记者

在 GitHub 中显示所有的测试结果， [test reporter](https://github.com/marketplace/actions/test-reporter) 动作有助于将代码和测试过程的重要部分放在一个地方。作为“检查运行”的一部分，提供 XML 或 JSON 格式的结果，这个动作用有用的统计数据告诉你代码在哪里失败了。

Test reporter 已经支持大多数流行的测试工具，例如。NET、JavaScript 等等。此外，你可以通过提出问题或贡献自己来增加更多内容。

支持的框架:

*   。NET: xUnit、NUnit 和 MSTest
*   飞镖:测试
*   颤振:测试
*   Java: JUnit
*   JavaScript: JEST 和 Mocha

## 2:构建和推送 docker 映像

顾名思义，[构建和推送 Docker 图像动作](https://github.com/marketplace/actions/build-and-push-docker-images)允许您构建和推送 Docker 图像。

使用 [Buildx](https://github.com/docker/buildx) 和[莫比 BuildKit](https://github.com/moby/buildkit) 特性，你可以创建多平台构建、测试映像、定制输入和输出等等。

查看动作页面的完整功能列表，包括[高级使用](https://github.com/marketplace/actions/build-and-push-docker-images#advanced-usage)和如何[定制它](https://github.com/marketplace/actions/build-and-push-docker-images#customizing)。

## 3:设置 PHP

[Setup PHP 动作](https://github.com/marketplace/actions/setup-php-action)允许你设置 PHP 扩展和。ini 文件，用于所有主要操作系统上的应用程序测试。

它还兼容 GitHub 的 composer、PHP-config、symfony 等工具。请访问市场页面，查看兼容工具的完整列表。

[GitTools 动作](https://github.com/marketplace/actions/gittools)允许你在管道中同时使用 [GitVersion](https://gitversion.net/) 和 [GitReleaseManager](https://github.com/GitTools/GitReleaseManager) 。

GitVersion 通过语义版本化(也称为“Semver”)帮助解决常见的版本化问题，以实现项目间的一致性。GitVersion 有助于避免重复，节省重建时间，等等。有益于 CI 的是，它创建了标记构建的版本号，并使变量对管道的其余部分可用。

同时，GitReleaseManager 自动创建、附加和发布可导出的发行说明。

如果你只需要 GitVersion 的版本控制，这个列表后面还有一个同名的的[替代动作。](#gitversion-action)

## 5:动作自动释放

一旦设置为对您选择的 GitHub 事件(比如提交到您的主分支)做出反应，[动作自动发布](https://github.com/marketplace/actions/automatic-releases)工作流可以:

*   自动上传资产
*   创建变更日志
*   创建新版本
*   将项目设置为“预发布”

## 6:存储库调度

[存储库分派动作](https://github.com/marketplace/actions/repository-dispatch)使得从‘存储库分派’事件触发动作变得更加容易。此外，它允许您触发和链接来自一个或多个存储库的操作。

你需要[创建一个个人访问令牌](https://docs.github.com/en/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token)来执行这个操作，因为默认情况下 GitHub 不支持它。

## 7:拉动预览

[PullPreview 动作](https://github.com/marketplace/actions/pullpreview)允许您通过构建代码评审的实时环境来预览拉请求。

当使用“pullpreview”标签进行拉取时，此操作会检出您的代码，并使用 Docker 和 Docker Compose 部署到 AWS Lightsail 实例。

这允许你像你的客户一样玩你的新功能，或者炫耀你的想法的工作模型。

它还承诺与您现有的工具兼容，并与 GitHub 完全集成。

你应该知道的唯一一件事是，如果将它用于商业产品，你需要购买许可证。

## 8:报告生成器

[ReportGenerator 动作](https://github.com/marketplace/actions/reportgenerator)可以将覆盖率报告中最有用的部分提取成更容易阅读的格式。它允许您读取 HTML、XML 等格式的数据，以及各种文本摘要和特定于语言的格式。

## 9: Git 版本

虽然有点像 GitTools 动作启用的 [GitVersion ***工具***](#git-tools) ，但是这个 [Git version](https://github.com/marketplace/actions/git-version) 本身就是一个动作。

然而，像外部工具一样，它提供了简单的 Semver 版本控制来帮助跟踪您的不同版本。如果您只想获得版本控制方面的帮助，而不需要 GitReleaseManager，这将非常有用。

## 10: GitHub 动作测试仪(github-action-tester)

github-action-tester 是一个让你启动 shell 脚本进行测试的动作。

安装完成后，只需将您的脚本添加到您的存储库中，然后用您需要的事件启动它们。

## 奖励回合:八达通

作为 GitHub 的官方技术合作伙伴，Octopus Deploy 已经在 GitHub 市场中[了 10 个经过验证的 Octopus 操作，使您的部署过程自动化、执行任务、创建包等等变得更加容易。](https://github.com/marketplace?query=octopus&type=actions&verification=verified_creator)

鉴于 GitHub 动作服务的本质，其他用户[也贡献了一些与章鱼相关的动作](https://github.com/marketplace?type=&verification=&query=Octopus+)。如果你想与 Octopus 进行更多的集成，可以看看这些。您还可以了解为什么我们推荐您[使用 GitHub 构建，使用 Octopus](https://octopus.com/github) 部署。

## 如何安装操作

在 GitHub 中安装动作很简单:

1.  在 [GitHub Marketplace](https://github.com/marketplace?type=actions) 上找到你想要的动作。
2.  阅读市场页面以检查先决条件。
3.  点击右上角的**使用最新版本**(或者根据需要选择旧版本)。
4.  从弹出窗口中复制代码，粘贴到您的存储库中。yml 文件，并保存。
5.  请务必阅读该操作的文稿，以检查任何额外的设置以及如何使用该操作。

## 接下来呢？

如果我们选择的行动不适合您的项目，或者您需要 CI 范围之外的东西，还有很多可供选择。通过 [GitHub 市场](https://github.com/marketplace?type=actions)搜索更多信息。

[试用我们免费的 GitHub Actions 工作流工具](https://oc.to/GithubActionsWorkflowGenerator)来帮助您快速为您的 GitHub Actions 部署生成可定制的工作流。

## 观看我们的 GitHub 行动网络研讨会

[https://www.youtube.com/embed/gLkAs_Cy5t4](https://www.youtube.com/embed/gLkAs_Cy5t4)

VIDEO

阅读我们的[持续集成系列](https://octopus.com/blog/tag/CI%20Series)的其余部分。

愉快的部署！