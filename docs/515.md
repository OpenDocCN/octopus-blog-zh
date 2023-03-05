# 空间-八达通部署 2019.1 -八达通部署

> 原文：<https://octopus.com/blog/octopus-release-2019.1>

[https://www.youtube.com/embed/9YztgS1wUmk](https://www.youtube.com/embed/9YztgS1wUmk)

VIDEO

## 关注对你来说重要的空间

我们很自豪能够提供[个空间](https://octopus.com/docs/administration/spaces)。我们对 Spaces 的目标是帮助团队更好地组织他们的 Octopus 服务器，并专注于对他们来说重要的项目、环境和部署。这将减少噪音，让团队更有效地工作。

## 在这篇文章中

## 给团队自己的空间

空间是一个简单的概念，对每个人都有很大的好处。它允许团队将项目、环境、租户、步骤模板和其他资源分组到一个空间中。

我们更新的导航栏让您可以在空间之间快速跳转。你不会看到数十或数百个不相关的项目和其他 Octopus 资源，你只会看到那个空间的资源。

## 通过改进的权限控制团队领导

作为构建空间的一部分，我们需要改进我们的安全性和权限。我们的安全模型现在更易于管理，团队可以独立管理其空间的安全性。这还可以消除向系统管理员提交请求的需要。

我们更改后的安全模型让您可以更轻松地管理空间或团队级别的访问控制。

## 重大变化

使用空间完全是自愿的。我们已经做了大量的工作和测试，以确保该功能尽可能向后兼容。

为了支持 Spaces 并改进你如何[配置团队](https://octopus.com/blog/team-configuration-improvements)我们做了一些突破性的 API 修改。在升级之前，你应该[在这里](https://octopus.com/downloads/compare?from=2018.12.1&to=2019.1.0)复习全套。

## 整合/触手

一旦你更新到 2019.1 并且想要开始使用你的 Octopus 实例中的空间特性，你将需要升级你的集成，包括[触手](https://octopus.com/downloads)。

## 升级

通常，升级 Octopus Deploy 的[步骤适用。更多信息请参见](https://octopus.com/docs/administration/upgrading)[发布说明](https://octopus.com/downloads/compare?to=2019.1.0)。

*   自托管八达通客户可以通过安装[八达通服务器 2019.1](https://octopus.com/downloads) 今天开始使用空间。注意`2019.1`是没有[长期支持](https://octopus.com/docs/administration/upgrading/long-term-support)的快车道发布。空间将包含在 2019 年底 Q1 下一次 [LTS](https://octopus.com/docs/administration/upgrading/long-term-support) 发布的章鱼中。

*   Octopus Cloud 客户将在其维护窗口的大约 2 周内开始接收最新的 bits。

这个月到此为止。欢迎给我们留下评论，让我们知道你的想法！前进并展开！

## 想了解更多

*   [阅读](https://octopus.com/docs/administration/spaces)所有关于空间的好处
*   [观看](https://hello.octopus.com/webinar-spaces-workers/on-demand?utm_referrer=http%3A%2F%2Foctopus.com%2Fblog%2Foctopus-release-2019.1)我们的网上研讨会“用空间和工人扩展你的章鱼”
*   [阅读](https://g.octopushq.com/spaces)空间文档
*   在我们的博客系列中阅读关于我们建造空间的旅程