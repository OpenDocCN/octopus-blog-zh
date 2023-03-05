# Octopus Deploy 3.5:订阅、Azure 广告、GoogleApps 和扩展性

> 原文：<https://octopus.com/blog/octopus-deploy-3.5>

虽然[多租户部署](http://docs.octopusdeploy.com/display/OD/Multi-tenant+deployments)和[弹性和瞬态环境](http://docs.octopusdeploy.com/display/OD/Elastic+and+Transient+Environments)一直是我们最近在 Octopus 的一大焦点，但在身份验证领域也发生了一些令人兴奋的事情，我们称之为[订阅](http://g.octopushq.com/Subscriptions)的新功能可以帮助您订阅活动，并在 Octopus 内发生事情时通过电子邮件或 webhooks 获得通知。

[下载章鱼 3.5](https://octopus.com/downloads)

我们删除了 Octopus 3.5.0 的下载链接，因为我们发布了一些错误，导致一些客户无法登录 Octopus 门户网站。请下载最新版本并使用它！

## 捐款

当我们引入[项目触发器](http://docs.octopusdeploy.com/display/OD/Project+Triggers)，特别是[自动部署触发器](http://docs.octopusdeploy.com/display/OD/Automatic+Deployment+Triggers)时，Octopus 在自动化领域迈出了另一大步。我们需要解决的一个问题是通知。

> 如果不登录 Octopus 并查看我的仪表盘，我如何知道我的自动部署(在凌晨 2 点我睡着时触发)实际上是成功还是失败？

介绍 [Octopus Subscriptions](http://g.octopushq.com/Subscriptions) ，这是一项新功能，允许您订阅 Octopus 中正在发生的事件。

你现在可以设置**电子邮件**或**网页挂钩**订阅，随时了解 Octopus 内发生的任何事件。一些明显的例子包括:

但是这还不是全部！您可能只想通过电子邮件收到与您团队环境相关的所有事件的摘要。或者，您可能想知道 Octopus 何时检测到您的机器离线或健康状况发生变化。您甚至可以使用 webhook 订阅来构建您自己的对 Octopus Deploy 中事件的自动响应。

Octopus Deploy 3.5 中的订阅服务完全由您想象:)

## OpenID 连接身份验证提供程序

到目前为止，Octopus 服务器有两种认证模式。一种是 it 自己处理身份管理，另一种是遵从 Active Directory。在 3.5 版中，我们增加了对两个 OpenID Connect 提供商的现成支持，即 **Azure AD** 和 **GoogleApps** 。

Octopus 身份验证的更新已经在雷达上出现了一段时间，我们收到了许多客户添加 Azure AD 和/或联合身份支持的请求。在 Octopus，我们使用 GoogleApps 进行身份管理，因此我们支持这两个提供商是有意义的。

了解更多关于我们的 [Azure AD](http://g.octopushq.com/AuthAzureAD) 和 [GoogleApps](http://g.octopushq.com/AuthGoogleApps) 支持。

## 多个身份验证提供者

除了支持 OpenID Connect 身份验证提供者，我们现在还支持同时启用多个身份验证提供者。也就是说，你可以启用 Azure AD、GoogleApps 和 UsernamePassword 提供程序。为了支持这一点，configure 命令中添加了一些新选项。

了解[配置认证提供者](http://docs.octopusdeploy.com/display/OD/Authentication+Providers)。

## 自动登录

当我们在这个领域时，另一个被请求的功能是启用自动用户登录的能力。这适用于只使用一个身份验证提供者的场景，并且不需要在 Octopus Deploy UI 中直接输入用户名/密码。即当表单验证被禁用时，OpenID 连接提供者或活动目录(域)提供者。

了解[自动登录](http://docs.octopusdeploy.com/display/OD/Authentication+Providers#AuthenticationProviders-AutoLogin)。

## Octopus HA 集群范围设置和 server.config

我们存储在 server.config XML 文件中的几个设置在过去曾在 Octopus HA 中引起过小问题，当时不同的节点无意中配置了冲突的设置。鉴于身份验证设置是原因之一，并且我们现在希望添加更多的身份验证设置，我们决定将它们移动到数据库中。

这些数据的迁移应该是完全透明的。升级到 3.5 后，当服务器重新启动时，它会自动发生。

**对于那些直接对 server.config XML 文件使用配置管理工具(如 Chef、Puppet 或 Desired State Configuration (DSC ))的人来说，如果您依赖于我们移动的值，这将是一个突破性的变化。**

我们希望今后对配置管理的更好支持将会抵消由此带来的任何不便。为此，我们向服务器添加了一个新的`show-configuration`命令。

此命令可用于将 XML(与旧的 server.config 兼容)导出到控制台或指定文件。还有两种支持导出的 JSON 格式，这使得在 PowerShell 和 Node.js 这样的语言中使用输出更好。

了解[显示配置命令](http://g.octopushq.com/ShowConfiguration)。

## Octopus 部署服务器可扩展性

事实证明，以一种在用户拥有的无数不同配置中都能很好工作的方式与 Active Directory 集成真的很难。将集成牢牢嵌入 Octopus 服务器也给我们和我们的用户带来了故障排除的困难，而且定制也是不可能的——直到现在！

我们已经改变了这一点，我们很高兴地宣布，章鱼部署服务器现在是可扩展的！

所有的 Active Directory 集成代码都被转移到一个扩展中，我们将提供开箱即用，但将保持开源。因此，如果您需要一个定制的实现来更好地适应您的环境，现在这一途径将变得简单。该项目可在 [Github](https://github.com/OctopusDeploy/DirectoryServicesAuthenticationProvider) 上获得

Azure AD 和 GoogleApps 的集成也是作为扩展构建的，并将是开源的。这两个项目由 [Github](https://github.com/OctopusDeploy/OpenIDConnectAuthenticationProviders) 上的一个 OpenID Connect 相关项目管理

了解[定制我们的扩展](http://g.octopushq.com/ServerExtensionsCustomising)或[编写自己的扩展](http://g.octopushq.com/ServerExtensionsAuthoring)。

我们很想知道这些新特性如何帮助您的部署场景，并期待您的反馈。

[下载章鱼 3.5](https://octopus.com/downloads)

愉快的部署！