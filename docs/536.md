# 八达通可能发布 3.13 -八达通部署

> 原文：<https://octopus.com/blog/octopus-release-3-13>

[![Octopus 3.13 release announcement](img/37d9520fd1875b0d39576143d787bdd9.png)](#)

本月的发布带来了一些令人兴奋的新功能，包括对 Azure Service Fabric、HSTS、可选生命周期和性能改进的支持，等等！

## 在这篇文章中

## 发布之旅

[https://www.youtube.com/embed/LljoIA8wtHQ](https://www.youtube.com/embed/LljoIA8wtHQ)

VIDEO

## Azure 服务结构支持简介

我们很高兴地宣布，Octopus 现在包括对部署 Azure 服务架构应用程序的一流支持。

自今年早些时候的 [RFC](https://octopus.com/blog/rfc-azure-service-fabric) 以来，我们一直忙于创建新的 step 模板来帮助您连接和部署您的服务结构集群应用程序。

这些新的服务结构步骤现在可以帮助您:

*   部署服务架构应用([了解更多信息](https://octopus.com/docs/deployments/azure/service-fabric/deploying-a-package-to-a-service-fabric-cluster)
*   运行服务结构 SDK PowerShell 脚本([了解更多信息](https://octopus.com/docs/deployments/custom-scripts/service-fabric-powershell-scripts)

这两个步骤都需要连接到集群。因此，我们包含了对安全模式的支持，包括不安全、客户端证书和 Azure Active Directory。

**服务架构 SDK**
由于服务架构的依赖性，您需要手动将[服务架构 SDK](https://g.octopushq.com/ServiceFabricSdkDownload) 安装到您的 Octopus 服务器上。然后，您可以开始使用 Octopus 来帮助编排您的服务结构应用程序部署。

## 想了解更多？

您可以从我们的主[部署到服务结构](https://octopus.com/docs/deployments/azure/service-fabric)文档中了解更多关于这些新特性的信息。

## HTTP 严格传输安全(HSTS)

HTTP 严格传输安全是一个 HTTP 头，可用于告诉 web 浏览器，即使用户尝试使用 HTTP，也只能使用 HTTPS 与网站通信。这可以大大减少您的攻击面，并且是安全专业人员经常推荐的。

我们现在可以按需发送这个头，但是由于存在一些潜在的复杂情况，它在默认情况下是不启用的。如果你有你的八达通服务器暴露在互联网上，我们建议[阅读并启用 HSTS](https://octopus.com/docs/how-to/expose-the-octopus-web-portal-over-https#HSTS) 。

## 可选的生命周期阶段

从我们的待办事项中去掉另一个高等级的[用户意见建议](https://octopusdeploy.uservoice.com/forums/170787-general/suggestions/8475958-lifecycle-optional-phase-or-optional-environment)，你现在可以在你的生命周期中创建可选的阶段，这些阶段可以在进展过程中跳过。这个特性将有助于您自由地将您的版本部署到一组环境中，而不会阻碍部署的继续。当这种行为是预先知道的，并且是标准发布管道的一部分时,[渠道](https://octopus.com/docs/deployments/patterns/branching)工作得很好，例如，总是将修补程序发布直接推送到 UAT，但是这种方法对于可选阶段功能带来的更灵活的规则集来说太死板了。

了解关于[可选生命周期阶段](https://octopus.com/docs/releases/lifecycles)的更多信息。

## 浏览器缓存

对于 Octopus 服务器来说，加载仪表板可能是一项数据密集型操作。它可能需要提取项目的所有版本和部署，对它们进行比较、排序和过滤，然后序列化并返回到浏览器进行渲染。在大型实例中，这可能需要几秒钟才能完成。每次客户端发出这个请求，都会占用服务器资源，而这些资源通常(考虑到目前大约每 5 秒钟更新一次)可能都没有改变。相反，门户现在将缓存一些高成本查询的响应，并且只在服务器上发生新事件时才重新加载数据。虽然可以禁用此功能，但预计这将使服务器对所有用户的响应更快。

## 脚本失败并显示消息

现在可以定制部署概述上的消息，请参考[脚本失败并显示消息](https://octopus.com/docs/deployments/custom-scripts/logging-messages-in-scripts)

## 修改任务状态

您是否曾经部署到生产环境中，却只有最后一步“电子邮件发布派对邀请！”失败？或者，您可能成功部署了，但在一些 QA 人员决定回滚后。现在，您可以修改任务的状态。当状态为`Success`、`Failed`或`Canceled`的任务完成时，您可以通过提供新的任务状态和更改原因，从任务屏幕编辑状态。提交后，任务状态将被更新，任务历史记录中的一个条目将包含一个带有更改的审计条目。执行此操作需要一个名为`TaskEdit`的新权限。默认情况下，`TaskEdit`权限只授予内置的管理员团队。

## 频道索引版本模板变量

例如

```
#{Octopus.Version.Channel[MyChannel].LastMajor} 
```

目前，当使用模板计算发布版本时，许多变量都是可用的。例如

```
#{Octopus.Version.LastMajor}
#{Octopus.Version.NextMajor}
#{Octopus.Version.LastMinor}
... 
```

当使用[通道](https://octopus.com/docs/releases/channels)时，相应的变量可用于*电流*通道。例如

```
#{Octopus.Version.Channel.LastMajor}
#{Octopus.Version.Channel.NextMajor}
#{Octopus.Version.Channel.LastMinor}
... 
```

缺失的部分是引用其他通道的*版本组件的能力(即不是正在创建的发布版本的通道)。*

例如，您的项目可能有通道:`Release`、`PreRelease`和`HotFix`。现在，您将能够定义一个版本模板，例如:

```
#{if IsRelease}
  #{Octopus.Version.Channel.LastMajor}.#{Octopus.Version.Channel.NextMinor}.0
#{/if}
#{if IsPreRelease}
  #{Octopus.Version.Channel[Release].LastMajor}.#{Octopus.Version.Channel[Release].NextMinor}.0#{Octopus.Version.Channel.NextSuffix}
#{/if}
#{if IsHotFix}
  #{Octopus.Version.Channel[Release].LastMajor}.#{Octopus.Version.Channel[Release].LastMinor}.#{Octopus.Version.Channel.NextPatch}
#{/if} 
```

使用通道索引变量，如`#{Octopus.Version.Channel[Release].LastMajor`允许您引用其他通道的上一\下一版本组件。

注意:这个模板依赖于定义通道范围的变量`IsRelease`、`IsPreRelease`和`IsHotFix`。

## 升级

升级 Octopus Deploy 的所有常规[步骤都适用。请注意，在这个版本中，围绕`show-configuration`命令有一个微小的突破性变化。更多信息请参见](https://octopus.com/docs/administration/upgrading)[发行说明](https://octopus.com/downloads/compare?to=3.13.0)。

## 包裹

这个月到此为止。我们希望你喜欢最新的功能和我们的新版本。欢迎给我们留下评论，让我们知道你的想法！愉快的部署！