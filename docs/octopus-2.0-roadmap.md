# Octopus 2.0 及以后的路线图- Octopus 部署

> 原文：<https://octopus.com/blog/octopus-2.0-roadmap>

我刚刚给我们的[邮件列表用户](http://eepurl.com/ft5-o)发了一封电子邮件，询问反馈和想要尝试 Octopus 2.0 的早期用户。如果你不在邮件列表上或者没有收到邮件，你可以[参加这里的简短调查](https://docs.google.com/a/paulstovell.com/forms/d/1aUvNTSByqwjLm7YmMSvUUShw69sPzVvw4M-ucvOsJ5o/viewform)。

## 2.0 的愿景

在[计划 Octopus 2.0](http://octopusdeploy.com/blog/octopus-2.0-vision) 中，我分享了一些我们对 2.0 版本的愿景:

> 我们不打算完全重写所有的东西，因为那是个坏主意。但与此同时，2.0 将会有所不同。1.0 是一个很好的方式来学习更多我们试图用 Octopus 解决的问题，2.0 是我们应用这些经验的地方。Octopus 2.0 将是其他人已经开发的产品。

我们希望在接下来的 6-12 个月内构建许多重要功能，但它们并不完全符合我们现有的架构。Octopus 的架构在 Octopus 1 上运行良好。但是当我们看着地平线上的章鱼 2。x 可能是，我们有许多内部变化要做，以支持他们中的许多人。自从 Octopus 第一次被设计出来，我们已经学到了很多关于自动化部署应该如何工作的知识，所以我们咬紧牙关，在过去的几个月里，我们一直忙于重构。我们所做的最大改变是:

*   重写我们的服务器/代理通信栈，以在 Mono 上工作并支持基于推和拉的部署
*   创建一个[强大而广泛的 RESTful API](https://github.com/OctopusDeploy/OctopusDeploy-Api/wiki)
*   重写我们的 UI，使其完全依赖于 REST API——当 2.0 发布时，你在 UI 中可以做的事情，都可以通过 REST API 来完成
*   对步骤和变量建模和存储方式的重大改变，让您更容易地在项目/环境之间共享它们

最初，我设想我们会做出这些改变，然后在这些改变的基础上添加一堆特性，并把它称为 2.0。相反，我们将:

1.  尽可能快地将我们所拥有的发送给一小部分人(这就是调查的目的)
2.  擦亮它
3.  释放它

一旦 2.0 稳定并发布，我们将尝试回到两周发布一次的节奏，一次添加一两个特性。

## 什么流行什么不流行？

我们的 Trello 板已经更新，让你更好地了解什么是在和什么是未来。这是 Octopus 2.0 中将要出现的内容:

**[章鱼 2.0 板](https://trello.com/b/iXxdTj9B/octopus-2-0)**

这是我们计划在 Octopus 2.0 发布后发布的公告板:

**[章鱼 2。x 板](https://trello.com/b/CylayR6o/octopus-future)**

Octopus 2.0 仍将是一个大版本，但它不会像我最初想象的那么大，这至少意味着我们将能够很快推出它。毕竟，不在真实用户手中的软件不会变得更好。