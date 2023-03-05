# 八达通 7 月发布 2018.7 -八达通部署

> 原文：<https://octopus.com/blog/octopus-release-2018.7>

[![Octopus Deploy 2018.7 release banner](img/d65c7508110170a26d26445f86d55956.png)](#)

这个月，我们的头条特色是 **Octopus Workers** ，它允许你将部署工作从你的 Octopus 服务器上转移出去。这种变化带来了几个好处，包括改进的安全性、更好的性能和其他很酷的东西，比如创建专用于特定任务的机器池的能力。

## 在这篇文章中

## 发布之旅

[https://www.youtube.com/embed/N4uBvgB3ehM](https://www.youtube.com/embed/N4uBvgB3ehM)

VIDEO

## 八达通工人和工人池

Octopus workers 和 worker pools 允许您将部署工作从 Octopus 服务器上转移出去。章鱼一直有一个内置的工人，但我们从来没有这样叫它。每当您在 Octopus 服务器上执行脚本的部署工作时，它都会使用集成工作器。这项工作包括 Azure 和 AWS 步骤，以及您指定在 Octopus 服务器上运行的脚本步骤。

随着工人和工人池的引入，您现在可以选择将这项工作转移到单独的机器上，并创建工人池以用于特殊目的。步骤的执行方式没有任何变化；工作人员提供了一个关于在哪里执行这些步骤的选项。

八达通工人带来许多好处，包括:

*   提高了安全性，因为自定义脚本不在你的 Octopus 服务器上运行。
*   为 Octopus 实例和项目部署提供更好的性能。
*   其他很酷的东西，如创建工作人员池来处理特定的任务，如云部署、数据库部署和任何其他特定的任务，在这些任务中，您可能会从一组专用的机器中受益。

在我们的[章鱼工人博客系列](https://octopus.com/blog/tag/Workers)中阅读更多相关内容。

### 批准

> 工人和工人池受许可证限制吗？它们有点像部署目标，对吗？

在这一点上，我们决定限制不同许可证类型的工作线程和工作线程池的使用:

1.  使用不受限制的`Community`(免费)、`Professional`、`Team`或`Enterprise`许可证的客户仅限一名员工。这使您能够从使用内置工作器过渡到[更安全的配置](https://octopus.com/docs/administration/security/hardening-octopus#configuring-workers)。
2.  使用不受限制的`High Availability`许可证的客户可以获得**无限制的**工人。
3.  拥有当前`Standard`或`Data Center`执照的客户还可以获得**无限量**工人。*这些许可证是在 Octopus 4.0 前后推出的。*
4.  [章鱼云](https://octopus.com/cloud)客户使用`Standard`订阅获得**无限**工人。
5.  [章鱼云](https://octopus.com/cloud)使用`Starter`订阅的客户仅限一名工作人员。毕竟只是为了入门。

随着时间的推移，我们可能会决定改变这些限制。如果你对你的执照以及它如何适用于工人有任何疑问，请联系 sales@octopus.com。

## 性能和抛光改进

我们还发布了一些性能和改进。也就是说，我们做了一些重要的更新来提高 Octopus 服务器的性能和可用性。特别是对于较大的安装，包括在某些情况下大大降低 SQL server 上的 CPU 使用率，对删除的改进，以及更快的项目和基础结构仪表板。我们一直在努力改善我们的用户体验，本月，我们调整了可变快照更新流程，并改进了生命周期、渠道和角色范围页面。

## Azure 网站和插槽

在这个版本中，我们恢复了对 **Azure 网站部署插槽**的一流支持，允许您为 **Azure Web App** 部署目标选择特定插槽，或者直接在**部署 Azure Web App** 步骤上设置插槽。我们还改进了 Azure 网站选择器的性能。

## 重大变化

列出 Azure 网站`\api\accounts\{id}\websites`的 API 端点不再返回部署槽，只返回生产网站。有一个新的 API 端点`/api/accounts/{id}/{resourceGroupName}/websites/{webSiteName}/slots`用于列出单个 Web 站点的部署位置。

## 已知问题

当使用[Dynamic infra structure PowerShell cmdlet](https://octopus.com/docs/infrastructure/dynamic-infrastructure/azure-web-app-target)创建新的 Azure PaaS 部署目标时，如果后续步骤从外部工作器向新创建的目标部署包，部署将失败。在脚本步骤和包部署步骤之间添加一个**健康检查**步骤，为**全面健康检查**配置，这将允许部署计划为工作进程获取必要的包。我们已经创建了一个 [GitHub 问题](https://github.com/OctopusDeploy/Issues/issues/4731)来跟踪这个问题，直到它得到解决。

## 升级

升级 Octopus Deploy 的[步骤](https://octopus.com/docs/administration/upgrading)照常适用。更多信息请参见[发行说明](https://octopus.com/downloads/compare?to=2018.7.0)。

## 包裹

这个月到此为止。欢迎给我们留下评论，让我们知道你的想法！前进并展开！