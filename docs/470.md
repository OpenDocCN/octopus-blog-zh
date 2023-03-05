# 章鱼。客户端开源- Octopus 部署

> 原文：<https://octopus.com/blog/octopus-client-goes-open-source>

![Octopus.Client goes Open Source](img/bbfaa8de8ad799ef202616e598fe8c61.png)章鱼。Client 是一个代码库，它使得从使用 [Octopus REST API](http://docs.octopusdeploy.com/display/OD/Octopus+REST+API) 变得容易。NET 应用程序、C#和 powershell 脚本以及 [LinqPad](https://www.linqpad.net/) 。可以通过 web 界面完成的任何事情也可以使用库以编程方式完成。

几周前我们分离了章鱼。客户端从我们的主要代码库，悄悄地把它的方式与开源许可证。它在我们的 [OctopusClients](https://github.com/OctopusDeploy/OctopusClients/tree/develop) GitHub 知识库中找到了自己的家。

要报告错误、提出问题或请求功能，请使用我们的[支持论坛](http://help.octopusdeploy.com/discussions)或给我们发电子邮件。我们会对问题和拉取请求进行监控，但是如果它与您遇到的问题有关，请通过我们的支持渠道联系我们，以便我们可以相应地对其进行优先级排序。此外，请首先与我们讨论任何大的变化，以确保它符合我们的产品目标。

## 取样器

我们还开源了一个工具，在 Octopus 实例中创建样本数据，用于测试和演示目的。您可以在 [Sampler](https://github.com/OctopusDeploy/Sampler) GitHub 资源库中找到源代码和文档，在资源库的[发布选项卡](https://github.com/OctopusDeploy/Sampler/releases)上可以找到发布。

## 服务器和兼容性

我们已经决定为我们的开源库和工具采用[语义版本](http://semver.org/)。这打破了我们之前保持版本号与 Octopus 服务器同步的惯例。我们的一些图书馆已经开始消失了。

章鱼的主要版本。客户端我们将保持向后兼容一套版本的八达通服务器，允许无忧未成年人和补丁升级。

相反，我们也尽可能保持我们的服务器 API 的向后兼容性，同时仍然能够添加新的特性。这意味着老版本的章鱼。客户端将与新版本的八达通服务器。

请参考我们的[兼容性](http://docs.octopusdeploy.com/display/OD/Compatibility)页面，了解哪些版本适用于特定版本的 Octopus 服务器。

## 。NET 核心和异步 API

我们还移植了章鱼。客户端到。NET Core，目标是 netstandard1.6 和。NET 框架 4.5。这意味着该库现在可以在任何地方使用。净核心运行。现已发布 4.0 版本，兼容八达通服务器 3.3 或更高版本。

由于。NET 标准版的 HttpClient 是完全异步的，我们现在只在。我们将重新添加一个同步 API，如果。NET 标准支持。

API 本身的结构在很大程度上没有改变，应该可以直接移植到异步 API。关于如何使用异步 API 的例子，请看我们的 [Octopus。客户文件](http://docs.octopusdeploy.com/display/OD/Octopus.Client)。

### 现有 API

如果您将该库与。NET Framework 中，同步 API 仍然可以保持向后兼容性。同步和异步 API 都具有完全相同的功能，并将继续如此。

我们还没有决定弃用同步 API，也不确定何时或是否会弃用。如果我们这样做，我们会给予充分的通知。

### Octo.exe

同样在 [OctopusClients](https://github.com/OctopusDeploy/OctopusClients) 资源库中的 Octo.exe 也已经被移植到。网芯。请密切关注近期的跨平台版本。有关该工具的更多信息，请参见我们的[文档](http://docs.octopusdeploy.com/display/OD/Octo.exe+Command+Line)。