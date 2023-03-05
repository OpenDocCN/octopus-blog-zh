# 如何全局使用 Octopus Deploy - Octopus Deploy

> 原文：<https://octopus.com/blog/io-global>

ioGlobal 是一家澳大利亚公司，为资源和环境行业提供应用地球化学、资源分析和数据系统自动化解决方案。他们在五月份开始使用 Octopus Deploy，并且 [Stacy Andrews](https://twitter.com/stacyandrews) 很友好地分享了他的经验。

**告诉我们你自己的情况**

Stacy Andrews，ioGlobal 私人有限公司，首席技术官。寻求开发商业软件的职业软件开发人员(主要在采矿业)。

您使用 Octopus 部署了哪些类型的应用程序？

我们主要使用 Octopus 部署 Web 应用程序。ASP.NET Web forms(版本 4)和 MVC(版本 3)以及相关的支持服务，它们是基于 OpenRasta 的 Web 服务(托管在 IIS 中)和基于 NServiceBus 的服务的组合。

你的环境是什么样的？

Octopus Deploy 目前正在管理以下系统的部署…

1.  我们有许多内部测试环境(在任何时候，我们都有 3 个以上的环境用于各种用途，每个环境有 3 个虚拟机(独立的应用服务器、数据库服务器和报告服务器)。

2.  我们有一个暂存环境，由北美的 RackSpace 托管。

3.  我们有两个生产环境。一个在澳大利亚，由 Amcom 主办。一个在北美，由 RackSpace 托管。

4.  我们有一个由 Amazon Web Services 托管的灾难恢复/预览环境。

我们有系统的现场托管版本；一家与 BHPB 合作，另一家与 Anglo Platinum 合作(由于防火墙/安全限制，这些公司目前没有使用 Octopus)。

**在使用 Octopus 之前，部署流程是怎样的？**

麻烦而复杂。我们将直接从内部构建服务器上运行的 powershell 脚本进行部署。这意味着我们在构建脚本中混合了部署问题(针对环境配置等)。

同样，我们必须为我们的生产环境手动运行这些脚本。

你认为 Octopus Deploy 的优势是什么？

其设置简单。一旦它开始运行，它就不会碍事了。

它使用了普遍可用的技术(NuGet、config transformation、powershell ),所以有大量的 doco。

你认为 Octopus Deploy 的弱点是什么？

如果您在错误的页面上，UI 中的一些工作流可能会令人困惑。

不能等待内置的持续部署工具(目前我们需要从 PS 脚本进行编排)。

你会对其他评估 Octopus Deploy 的人说些什么？

如果您从没有自动部署到使用 Octopus Deploy 慢慢来，慢慢干。从 web 应用程序开始(Octopus 让它非常容易部署)。

通读文档；它写得很好，有许多常识性的建议。

如果你有困难，可以看看 ioGlobal 的设置！