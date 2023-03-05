# 包装为。NETCore，打开。部署了 Octopus - Octopus 网络核心

> 原文：<https://octopus.com/blog/octopus-and-netcore>

[![Octopus Packaging .NET Core banner](img/517e75472bc116409205b83b55e968b8.png)](#)

自从这篇文章首次发表以来，我们已经重新命名了 Octo.exe，现在它是 Octopus CLI，更多信息，请参见这篇文章: [Octopus release 2020.1](https://www.octopus.com/blog/octopus-release-2020-1) 。

在过去的几个月里，我们收到了许多关于构建和打包的问题和请求。NET 核心应用程序。我们对此已经有一段时间的支持，但有趣的是支持建筑的请求数量。NET 核心应用程序。网芯。这到底是什么意思？意思是支持建筑。NET 核心应用程序。NET 核心，而不是完整的。NET 框架。想想 Linux 或 Mac OS 机器。

如果这是你正在工作或打算进入的领域，我们有一些令人兴奋的消息。除了 [2018.7](https://octopus.com/blog/octopus-release-2018.7) 中包含的所有其他令人兴奋的东西，我们已经更新了`octo.exe`，所以你现在可以通过. NET 命令行扩展来访问它。

## dotnet octo 简介

一段时间以来，`octo.exe`一直是跨平台的，支持在两个平台上运行。NET 框架和。网芯。然而，在. NET Core only 平台上进行构建时，执行`octo.exe`会很尴尬。

进入`dotnet octo`全局工具。它做 [`octo.exe`](https://octopus.com/docs/octopus-rest-api/octopus-cli) 所做的一切，但它可以被称为使用`dotnet octo <command>`。这提供了一种便捷的方式将`octo.exe`安装到任何安装了最新 dotnet SDK 版本的机器上。

要施展魔法，使用以下命令将`octo.exe`召唤到你的构建机器上:

```
dotnet tool install Octopus.DotNet.Cli --tool-path /path/to/install 
```

然后将`octo.exe`从沉睡中唤醒，召唤经典咒语，如`dotnet octo pack`和`dotnet octo create-release`。

正如对`tool-path`参数的说明，您可以用- global 标志替换它，它将全局安装该工具。根据您的构建机器配置，全局安装可能不是一个好主意，但是本地安装到运行构建的文件夹提供了更好的隔离。不幸的是，这种隔离带来了一个小问题，dotnet 命令行不提供相同的参数来帮助找到您放入自定义刀具路径的刀具。为了解决这个问题，您必须**确保将路径添加到环境路径变量**中。

还要注意，上面的命令将最新版本传到您的构建机器上。如果您想要更精细的控制，有一个[版本开关](https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-tool-install)。

那么，将所有这些魔法结合在一起的构建脚本会是什么样子呢？下面是一个简化的例子:

```
dotnet publish MyAwesomeWebApp -o myMarshallingFolder

octo pack --id=MyAwesomeWebApp --version=1.0.0.0 --outFolder=myArtifactsFolder --basePath=myMarshallingFolder

octo push --package=myArtifactsFolder\MyAwesomeWebApp.1.0.0.0.nupkg --server=https://my.octopus.url --apiKey API-XXXXXXXXXXXXXXXX 
```

就像我说的，这是简化的。当您从您最喜欢的构建工具中设置它时，您可能希望将它分成三个独立的步骤。

## 运行时和 SDK 要求

为了使用这里描述的 Octo，您必须使用。NET Core SDK `2.1.300`或更新版本。

## 构建服务器

我们的 TeamCity 扩展已经可以处理`octo.exe`和`dotnet octo`之间的切换，所以你不需要为`push`、`create-release`等改变你现有的步骤。你必须为`dotnet tool`命令制定一个策略。您可以在构建过程开始时将其作为脚本运行，或者在构建代理上预先运行。TeamCity 扩展很快也会推出一个单独的`pack`步骤，针对那些想要打包然后使用除 Octopus 内置饲料之外的饲料(例如 TeamCity 的饲料或 Artifactory)的人。

VSTS 扩展的 **v3.0** 更新包括支持使用`dotnet octo`的更新。这些变化包括不再使用 PowerShell，这使得它与运行 Linux 等操作系统的构建代理兼容。

## 但是 OctoPack 呢？

在这篇文章中还有一件事要讲。房间里的大象，如果你愿意的话。OctoPack 怎么样？

简而言之，OctoPack 依赖于 NuGet 和 MSBuild 的一些机制，这些机制在。NETCore world 并试图将其移植到这个新世界中似乎不会比使用`octo.exe`提供价值。

这里的一个关键部分是我们现在支持的应用程序格式。当 OctoPack 出现时，它有两种主要的应用程序类型需要担心。Web 应用和 Windows 应用。这两者都可以通过简单地抓取二进制输出和任何标记为内容的文件来打包(这就是 OctoPack 在构建 nuspec 文件时在内部所做的)。

快进到今天，我们有像云服务和服务结构这样的应用格式。在 OctoPack 中支持这些不断增长的格式是不现实的，所以我们建议使用 VS/MSBuild 中内置的`package`目标。在我们的文档中有一个如何做到这一点的例子。

对此需要注意的一点是，`octo.exe`使用`Octopus.Client`作为它的`push`命令，所以它仅限于推送到 Octopus 内置提要。如果您需要推送另一个套餐服务，您将需要使用`NuGet.exe`而不是`octo.exe`。我们正在寻找解决这个问题的方法。

## 结论

的。网络核心世界仍然是一个快速发展的地方，所以这是我确信将是一个更长的旅程中的一步。如果你在建造。NET 核心应用程序，请给`dotnet octo`一个旋转，并在下面给我们反馈，以帮助指导这个旅程。

## 了解更多信息