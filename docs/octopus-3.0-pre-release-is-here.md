# Octopus 3.0 预发布已经发布- Octopus 部署

> 原文：<https://octopus.com/blog/octopus-3.0-pre-release-is-here>

因此，我们一直在等待的大消息，[章鱼 3.0 预发布](http://octopusdeploy.com/downloads/3.0.0)现在向公众发布！

我们知道我们花了比预期更长的时间，这是一个巨大的变化。这篇文章会给你一个主要的概述。

对于 TL；博士，仔细读下一点。

这是一个预发布版本，这意味着它不是 Octopus Deploy 的官方支持版本(还有几周时间)。这意味着下载它，安装它，使用它，并通过给我们反馈和让我们知道您可能发现的任何错误来帮助我们改进它。但它可能并不完美，也不是最终版本。

这也意味着我们以不同的方式对待反馈和错误报告。

## 这很重要。

对于 Octopus Deploy 3.0 预发布支持，反馈和错误报告请使用我们新的[社区论坛](http://community.octopusdeploy.com/)。我们在那里张贴了很多，所以这是获得所有最新信息的地方。我们有一些如何张贴在那里，最终将成为博客帖子或文档的一部分。您还可以订阅预发布论坛，以便在有新活动时收到电子邮件。我们将在下载页面发布大部分版本，但我们可能会将一些临时版本直接发布到论坛。

两个人的。*支持，使用我们的常规支持渠道([support@octopusdeploy.com](mailto:support@octopusdeploy.com)和[我们的支持论坛](http://help.octopusdeploy.com/home))。

好了，关于细节，您可以阅读或观看我们的网络研讨会视频，更直观地了解新内容。

[https://www.youtube.com/embed/eUT2Vtw3L54](https://www.youtube.com/embed/eUT2Vtw3L54)

VIDEO

## 体系结构

在 3.0 中，我们花了很多时间，因为我们希望在性能和可支持性方面对架构进行一些更改。

### SQL Server

这是你们期待已久的大日子。从 3.0 开始，我们已经从 [RavenDB 迁移到 SQL Server](http://octopusdeploy.com/blog/tag/ravendb) 作为数据存储。我们有几篇博文解释了这一变化的原因和方式，你[可以阅读](http://octopusdeploy.com/blog/tag/ravendb)。这样做的结果是一个更快的产品，一个可伸缩性更好的产品，一个我们更容易支持的产品。这是以我们的安装体验为代价的，我们选择不捆绑嵌入式 SQL Server 实例，但我们也知道 SQL Server 安装对我们的客户来说是非常熟悉的事情。[我们支持 SQL Server 2008 及更高版本](http://docs.octopusdeploy.com/display/OD3/SQL+Server+Database+Requirements)和 Express edition，以及企业版和 Azure SQL 实例。

### 大比目鱼，我们的新通信栈

在章鱼 2 里。*我们使用了一个名为 Pipefish 的库，这是一个类似于 Akka.NET 的 Actor 框架，用于我们的通信。在 3.0 中，我们用我们称之为[大比目鱼](http://octopusdeploy.com/blog/tag/halibut)的东西取代了它，这简化了我们的代码库，也给了我们更容易跟踪和调试的东西，这意味着更容易支持。如果你想去看看它是如何工作的，它也是[开源的](https://github.com/OctopusDeploy/Halibut)。

## 表演

除了我们看到的这两个变化在开发成本和可支持性方面的收益之外，它们还使 Octopus [速度更快，可伸缩性更强。我们将在未来几周与您分享更多的指标，因为我们会进一步推动它。](http://octopusdeploy.com/blog/tag/performance)

### 鱿鱼

在 3.0 中，我们还改变了触手架构。触须用来包含所有的代码来完成包的部署，运行我们的约定(例如，改变配置文件等)以及完成像 IIS 配置这样的任务。这使得它与 Octopus 版本紧密耦合。我们现在把它分成两部分。触手是一个代理，除了安全地传输文件和运行命令之外，它什么也不做，所有的逻辑都在 [Calamari](http://octopusdeploy.com/blog/tag/calamari) 中。较新的卡拉马里软件包作为健康检查过程的一部分被推送到触手，所以这意味着触手升级较少。Calamari 也是[开源](https://github.com/OctopusDeploy/Calamari)。这对您来说意味着您可以针对您的环境派生和调整它，并拥有一些您自己的部署代码。

## 新功能

我们最终将 [Delta 压缩库](http://octopusdeploy.com/blog/tag/delta%20compression)添加到 Octopus 中用于包传输。根据 Nuget 包的内容，您应该会看到需要在版本之间传输到触手的数据量显著减少。对于许多客户来说，这将大大节省时间和网络利用率。

## 部署目标

我们在过去已经谈过一些关于[部署目标](http://octopusdeploy.com/blog/tag/deployment%20targets)的内容，但是我们已经把“机器”的概念变得更加通用了。

### 离线部署

离线部署是一个新的部署目标。很多人一直在寻求的。如果您是我们的客户之一，并且目前拥有用于部署到完全隔离的环境的自定义安装脚本，那么您现在可以添加一个离线部署目标。当您部署到这里时，我们将打包 Nuget 包、一个 Calamari 副本、一个包含发布变量的 JSON 文件和一个可以在目标机器上运行的引导脚本。这里有一个[“入门”](http://community.octopusdeploy.com/t/offline-deployments/40/8)的帖子，如果你想用的话可以看看。

这是把乌贼和触手分开的原因之一。现在，我们可以将所有的部署工具打包成这个包的一部分，您可以随身携带！

### Azure 网站

以前，我们的[库](https://library.octopusdeploy.com/#!/listing)中有一个 WebDeploy 脚本，它使用 Web Deploy 将包的内容推送到 Azure 网站。这现在作为部署目标内置。

### 云服务

我们还改进了对云服务的支持。作为其中的一部分，您现在可以对您的包执行任何约定(变量、配置转换等),因为我们将解包您的。cspkg，运行约定，然后在推送到 Azure 之前重新打包。

关于我们 Azure 支持的详细信息，[可以看看这个帖子](http://community.octopusdeploy.com/t/azure-integration-in-octopus-deploy-3-0-an-overview/73)。

### SSH 部署

我们已经关注 SSH 和 Linux 支持很久了，现在它就在这里！为此，我们让卡拉马里与[单声道](http://www.mono-project.com/)兼容，所以你需要在你的目标机器上安装。我们这样做是为了维护这些工具的单一代码库，并且 Linux 支持不会成为二等公民。此外，现在触手只是一个代理，这意味着我们两个平台的架构非常相似。要阅读更多关于我们的 SSH 支持[的内容，请阅读这篇文章](http://community.octopusdeploy.com/t/introducing-the-ssh-endpoint/70)，为了浏览一下如何设置自己的[，我们还有另一个](http://community.octopusdeploy.com/t/my-first-ssh-endpoint/71)。

## 移居者

[迁移器特性](http://octopusdeploy.com/blog/tag/migrator)是 Octopus 服务器管理器的一部分，也可以从命令行获得。该工具将:

*   将 2.6 版本的备份导入 3.0 版本
*   将 3.0 项目配置导出到 JSON 文件目录中
*   将 3.0 JSON 文件导入 Octopus Deploy

这将帮助您将现有的 Octopus 配置升级到 3.0。它将帮助您将 3.0 POC 或试运行过渡到生产环境。通过一些脚本，您还可以构建一些东西，从 staging Octopus 实例中导出项目，并将它们推送到生产实例中。

通过一些脚本，您还可以...(要说出令人吃惊或高兴的事情)听着...将您的整个 Octopus 部署配置提交到 Git(或者您选择的 VCS)中！！！！！！

## 一些你看不到的东西

你可能会注意到 Octopus 3.0 目前看起来和 2.6 很像。当我们发布最终版本时，这种情况将会改变。我们将在接下来的几周内对它的外观和感觉进行一些改动，但我们希望能推出一些功能供您测试。

您可能还会注意到，FTP 没有作为部署目标。FTP 是我们很难支持的另一个项目。我们使用的组件在某些环境下有效，而在其他环境下无效。由于使用 FTP 步骤的人非常少(我们认为他们中的很多人都是为 Azure 网站服务的)，我们决定不将该功能移植到 3.0。您仍然可以用 PowerShell 步骤实现相同的目标，我们可以考虑在库中提供一个。

如果这是你的一个问题，请让我们知道，我们会帮助你。

你可能会遇到的另一件事是，因为我们已经完全改进了我们的云服务支持，旧的 Azure Web 角色支持不是 migrator 将带来的东西。如果你在 2.6 中使用这个特性，你需要做一些手工移植。

## 所以让我们开始吧

正如我在文章开头所说，社区论坛是获取更多信息、反馈、错误报告和讨论的地方。任何章鱼 2。*应通过正常渠道提供支持。

一旦我们退出预发布，这种情况将会改变，我们希望社区论坛作为一个讨论更多一般事情的地方继续存在。因此，去抓住位，并开始！

## 关于许可的快速说明

有几个人问过这个问题。如果您与我们签订了有效的维护协议(即，您在过去 12 个月内购买了 Octopus 或更新了维护协议),则 3.0 是免费升级。如果您的维护已过期，您需要续订才能升级。

愉快的部署！