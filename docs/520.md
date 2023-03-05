# 自动发布说明和工作项目跟踪-Octopus Deploy 2019.4-Octopus Deploy

> 原文：<https://octopus.com/blog/octopus-release-2019.4>

我们将推出 Octopus Deploy 2019.4，重点是增强您的 CI/CD 渠道中的反馈循环。如果您曾经想要通过构建和部署更好地跟踪您的需求，这个更新是为您准备的。

此版本包括一些重要的更新:

## 集成

为了充分利用这个版本的特性，你需要更新你的 Octopus Deploy build 服务器插件并安装 Octopus Deploy 吉拉插件。

注意:我们也在更新我们的 Azure DevOps web 扩展，应该很快就可以使用了。

## 重大变化

由`Octopus.Server.exe` `show-configuration`命令返回的输出格式有一些非常细微的变化。这不太可能影响您，但是如果您正在使用它来驱动自动化，请在升级之前测试新版本。

## 升级

像往常一样，请遵循升级 Octopus Deploy 的[正常步骤。更多信息请参见](https://octopus.com/docs/administration/upgrading)[发布说明](https://octopus.com/downloads/compare?to=2019.4.0)。

*   自托管八达通客户今天就可以通过安装[八达通服务器 2019.4](https://octopus.com/downloads) 开始使用这些功能。注意`2019.4`是没有[长期支持](https://octopus.com/docs/administration/upgrading/long-term-support)的快车道发布。该功能集将包含在 2019 年 Q2 年底下一次 [LTS](https://octopus.com/docs/administration/upgrading/long-term-support) 发布的 Octopus 中。

*   Octopus Cloud 客户将在其维护窗口的大约 2 周内开始接收最新的 bits。

## 包裹

这个月到此为止。欢迎给我们留下评论，让我们知道你的想法！愉快的部署！