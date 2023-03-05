# 通过 TFS 团队构建和 Octopus Deploy - Octopus Deploy 实现自动化部署

> 原文：<https://octopus.com/blog/using-octopus-and-tfs-builds>

![Article date 2012-07-16 Octopus Version 1.X supported for this article](img/ba22567b541618dc18f50fae315515ae.png)

Damian Karzon 有一个很好的演示，展示了如何使用来自 TFS 团队构建的 Octopus Deploy:

> 我最近一直在玩 Octopus Deploy，我已经设置了我们的 TFS 构建服务器来自动创建版本并部署到开发环境中。下面是如何设置 TFS 构建服务器来创建 NuGet 包，以便通过 Octopus 进行部署。

Dave Welling 也有几篇文章[扩展了 TFS 的工作流程来发布一个包并将其部署到 Octopus](http://theredcircuit.com/2012/01/18/tfs-build-with-octopus-deploy-part-1/ "Octopus Deploy and TFS Workflow") :

> 为了让其他人不那么心痛，我会在下一篇博文中发布一些我自己的有用花絮。我创建了一个团队构建代码活动来启动 Octopus 版本的创建和部署。我已经创建了另一个，它将监控部署并把结果合并到您的构建中。

最后，Racingcow 有一个帖子分享了他的经历[让 OctoPack 和 TFS 一起玩](http://racingcow.wordpress.com/2012/05/07/octopus-and-tfs/ "OctoPack and TFS"):

> 总的来说，章鱼看起来真的很有前途。虽然与 TFS 融合本可以更容易，但我不确定这是八达通本身的错。也许一点额外的关于如何与 TFS 集成的文档(对于 NuGet dummies)会加速我解决我遇到的一些问题。然而，除此之外，Octopus 绝对是我向主要使用微软技术的商店推荐的东西。在我看来，它比使用 TFS 构建和 PowerShell 脚本更易于管理，而且肯定比手动部署要好得多。

我喜欢这些帖子的一点是，每个帖子都展示了不同的整合方式；Damian 使用定制的 MSBuild 目标文件，Dave 使用修改过的工作流(听起来很吓人！)而 Racingcow 依赖于对 OctoPack 的一点修改(我将把它合并到源代码中)。如果你正在考虑章鱼和 TFS，这些帖子绝对值得一读！