# GitHub 的操作与传统的构建服务器不同——Octopus Deploy

> 原文：<https://octopus.com/blog/githubactions-different-build-servers>

GitHub Actions 对于自动化和持续集成(CI)领域来说相对较新。GitHub Actions 提供“CI 即服务”,与传统的竞争平台有许多不同之处。

在这篇文章中，我们将探讨 GitHub 动作和传统构建服务器之间的区别。我们还要看看 GitHub Actions 是否是构建和测试代码的合适选择。

## GitHub Actions CI 已经内置在 GitHub 中

GitHub Actions 相对于传统 CI 平台的最大优势之一就是它的交付。GitHub Actions 是“CI 即服务”，已经包含在所有客户的每个存储库中(稍后将详细介绍定价)。

简单来说:如果你有一个 GitHub 的账户，你可以使用 GitHub 的动作。

如果你喜欢把所有的工作都放在一个地方，不用担心太多的移动部件，这就很有吸引力。

## 如果你不使用 GitHub，动作不是一个选项

您使用 GitHub 动作的能力取决于您或您的公司使用 GitHub。如果您使用其他代码库，或者您在本地托管您的代码，GitHub Actions 不是一个选项。

然而，值得一提的是，其他代码库服务，如 GitLab 和 BitBucket，也发布了自己的 CI 服务。如果其他服务更合你的口味，你可能还有其他选择。

## CI 不需要硬件(除非您愿意)

默认情况下，动作使用“跑步者”来完成工作流的作业。运行者是 GitHub 托管的虚拟机，使用 Windows、Linux 和 MacOS。这意味着您不需要维护自己的基础架构来执行 CI。不需要容纳硬件，不需要维护操作系统，不需要安装、更新或修补。GitHub 会为您处理这一切。

如果你需要 CI 的特定设置，而 GitHub 的跑步者不能胜任这项任务，或者你想要更多的控制，你也可以[自己主持你自己的跑步者](https://docs.github.com/en/actions/hosting-your-own-runners)。

## 许可和定价

传统的构建服务器通常是开源的，或者根据用户或实例的数量根据您的需求进行扩展。然而，CI 即服务意味着提供商可以在定价模式上有所创新。

例如，GitHub Actions[采用现收现付的方式](https://docs.github.com/en/billing/managing-billing-for-github-actions/about-billing-for-github-actions)，以运行时间来衡量所有工作。

所有用户每月都可以获得免费的工作分钟数和存储空间，其数量随您的计划而定。如果你超过了你的免费分钟数，你将被按分钟计费，并根据跑步者工作所需的操作系统进行加权。

GitHub Actions 对那些拥有公共回购或使用自己托管的跑步者完全免费。

## github 操作如何与事件和工作流自动化

GitHub Actions 使用您的 GitHub repo 中的活动(或外部事件，如果您使用'[repository dispatch event](https://docs.github.com/en/actions/using-workflows/events-that-trigger-workflows#repository_dispatch)' web hook)来触发工作流。您可以选择用单个或多个事件启动工作流，使用计划，或手动启动它们。

工作流的结构如下:

*   每个工作流都包含作业
*   每个作业都有在自己的运行器上执行的步骤
*   步骤是每个作业中的单个操作

例如，您可以创建一个工作流:

1.  每当有人创建拉请求时，测试您的代码
2.  成功测试后打包代码，并将其推送到您的部署工具

在这种情况下，工作流将有 2 个单独的作业。如果对作业 1 的测试成功，将触发作业 2。这两项工作都有自己的步骤，并在单独、干净的滑道上运行。

感谢 [GitHub Marketplace](https://github.com/marketplace) ，你可能根本不需要创建工作流。虽然不是 GitHub Actions 独有的，但是有数以千计的社区制作的动作和工作流可以用来实现您的需求。

Octopus 还创建了一个[工具来帮助你构建 GitHub Actions 工作流程](https://githubactionworkflows.com/)。如果您不确定从哪里开始，请尝试一下。

有关 GitHub 操作中工作流的更多信息，请查看:

## 如何安装操作

与您在大多数其他 CI 平台中添加和管理插件的方式相比，安装操作也有所不同。

例如，在 Jenkins，它们几乎是模块化的迷你应用程序。使用安装。hpi 包，一个 Jenkins 插件可以极大地影响外观、感觉和你在 Jenkins 实例中看到的选项。

在 GitHub 中安装动作包括从动作的 marketplace 页面复制代码，并将其粘贴到您的存储库中。yml 文件。动作不提供可见的变化，而是只提供新的功能，由动作自身的 repo 中的事件触发。

## 结论

这篇文章讨论了 GitHub Actions(以及 CI 即服务的概念)和传统 CI 平台之间的主要区别。

GitHub Actions 是一个理想的解决方案，如果你:

*   是 CI 概念的新成员
*   不想花时间设置和维护专用的构建服务器
*   不想或不需要在本地托管您的硬件或数据
*   像你所有的工具一样放在一个地方
*   已经使用 GitHub

您还可以了解更多关于使用 GitHub 构建[和使用 Octopus](https://octopus.com/github) 部署的信息，并在 GitHub Marketplace 中使用我们的[验证操作。](https://github.com/marketplace?query=octopus&type=actions&verification=verified_creator)

愉快的部署！