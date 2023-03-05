# 征求意见-从 scriptcs 迁移到 dotnet-script - Octopus 部署

> 原文：<https://octopus.com/blog/rfc-migrate-scriptcs-dotnet-script>

我们收到了[客户反馈](https://help.octopus.com/t/consider-use-dotnet-script-vs-scriptcs/22144)和[用户声音投票](https://octopusdeploy.uservoice.com/forums/170787-general/suggestions/31454668-allow-the-use-of-c-script-csx-using-net-core)，要求我们更新 Octopus 用来运行 C#脚本的工具，从 [scriptcs](https://github.com/scriptcs/scriptcs) 到 [dotnet-script](https://github.com/filipw/dotnet-script) 。这将:

*   在部署脚本中解锁较新的 C#语言功能
*   允许从脚本中直接引用 NuGet 包
*   不再需要安装 Mono 来在 Linux 部署目标上运行 C#脚本

C#脚本占了我们脚本步骤的大约 5%,所以我们想了解这个变化对我们用户的影响。

如果您在部署过程中使用 C#脚本，并且使用 SSH 和 Mono 部署到 Linux 目标，或者部署到运行早于 2012 年 R2 版的 Windows 版本的 Windows Tentacle 目标，建议的更改可能会影响您。

这篇文章概述了潜在的变化，以及迁移到 dotnet-script 和弃用 scriptcs 的权衡。我们还创建了一个 [GitHub 问题](https://github.com/OctopusDeploy/StepsFeedback/issues/9)，您可以在这里提供反馈，我们可以进一步评估对该功能的需求。

## 我们建议如何支持 dotnet-script

该意见征询书(RFC)提议移除`scriptcs`以支持`dotnet-script`。

为了将软件部署到您的服务器上，我们使用了[触手](https://github.com/OctopusDeploy/OctopusTentacle)，这是一个轻量级服务，负责与 Octopus 服务器通信，并调用[卡拉马里](https://github.com/OctopusDeploy/Calamari)。Calamari 是一个命令行工具，它知道如何执行部署，并且是所有部署操作(包括脚本执行)的宿主进程。我们目前为。NET Framework 4.0.0、4.5.2 和 netcore3.1。根据您的服务器操作系统、体系结构和版本，触手会接收这些 Calamari 版本之一。

历史上，Calamari 要求在 Linux 目标上安装 Mono 来执行`scriptcs`,因为它是完全编译的。NET 框架。随着跨平台的引入。使用 netcore3.1 Linux 的网络应用程序现在可以本地运行。NET 应用程序消除了 Mono 的复杂性和开销。Linux 目标目前默认接收 netcore3.1 Calamari，除了 [Linux SSH 目标](https://octopus.com/docs/infrastructure/deployment-targets/linux/ssh-target#add-an-ssh-connection)，它可以指定在 Mono 上运行脚本。

是基于. NET 的 C#脚本的现代实现。它可以在所有支持的目标上运行。网络应用程序(netcore3.1 及更新版本)。如果我们做出这一更改，这将意味着 C#脚本将只能在支持. NET 的目标上运行。Windows Server 2012 R2 和早期版本仅支持。所以这些目标将失去运行 C#脚本的能力。

## 影响

### 增加的功能

| 特征 | scriptcs | 点网脚本 |
| --- | --- | --- |
| C#版本 | 5 | 8 |
| 移除 Linux 对 Mono 的依赖 | ❌ | ✅ |
| 获取导入支持 | ❌ | ✅ |
| 允许未来。NET 5 和 6 支持 | ❌ | ✅ |

### 拟议方法的好处

所有包含在第 8 版之前的 C#语言特性现在都可以在你的 C#脚本中使用了。

消除对 Mono 执行脚本的依赖使我们与现代的跨平台相结合。NET 功能降低了调用 Mono 和相关问题的复杂性。

来自`dotnet-script`的 NuGet 导入支持允许在脚本中直接引用 NuGet 包，而不必[在脚本包](https://octopus.com/docs/octopus-rest-api/octopus.client/using-client-in-octopus)中包含 dll。新方法如下所示。

```
#r "nuget: RestSharp, 108.0.1"

using RestSharp;

var client = new RestClient("https://pokeapi.co/api/v2/");
var request = new RestRequest("pokemon/ditto");
var response = await client.ExecuteGetAsync(request);
Console.WriteLine(response.Content); 
```

### 使用 Mono 的 Linux SSH 目标

这种变化的一个代价是，在使用 SSH 和 Mono 的 Linux 部署目标上，C#脚本不再可用。

#### 移民

要针对 SSH linux 目标运行 C#脚本，您需要重新配置 SSH 目标，以使用通过 netcore3.1 运行的独立的 Calamari。

为此，[在您的 SSH 目标上选择自包含的 Calamari 目标运行时](https://octopus.com/docs/infrastructure/deployment-targets/linux/ssh-target#self-contained-calamari)。使用 Linux 触手的目标将继续像以前一样工作。

### Windows Server 2012 R2 版(及更早版本)目标

这一改变的另一个代价是`dotnet-script`只适用于 netcore3.1 及以上版本。这将使 C#脚本不可用于针对安装在早于 2012 年 R2 版的 Windows 上的 Windows Tentacles 的部署，因为这些版本正在运行。Calamari 的. NET 框架构建。

#### 工作区

我们开发了一个解决方法，因此您可以在受影响的 Windows 目标上继续使用 scriptcs，但是您必须更新您的部署过程。

1.  添加 [scriptcs NuGet 包](https://www.nuget.org/packages/scriptcs/)作为[引用包](https://octopus.com/blog/script-step-packages)。
2.  将 C#脚本的主体复制到下面 PowerShell 模板中的`$ScriptContent`变量中。

C#脚本中使用的任何参数都需要通过 scriptcs 参数传递，并在 ScriptContent 中使用`Env.ScriptArgs[Index]`格式引用。下面的模板显示了如何为`Octopus.Deployment.Id`执行此操作的示例。

```
$ScriptContent = @"
Console.WriteLine(Env.ScriptArgs[0]);
"@

New-Item -Path . -Name "ScriptFile.csx" -ItemType "file" -Value $ScriptContent

$scriptCs = Join-Path $OctopusParameters["Octopus.Action.Package[scriptcs].ExtractedPath"] "tools/scriptcs.exe"

& $scriptCs ScriptFile.csx -- $OctopusParameters["Octopus.Deployment.Id"] 
```

## 这个什么时候发布？

我们仍在评估这一变化可能会影响多少用户。在我们清楚了解我们将影响谁以及他们需要采取什么行动之前，我们不会做出或发布提议的变更。

## 我们需要您的反馈

我们仍在考虑这一变更，因此现在正是利用您的反馈来帮助制定这一提案的大好时机。我们制作了一期 [GitHub 来捕捉讨论](https://github.com/OctopusDeploy/StepsFeedback/issues/9)。

具体来说，我们想知道:

*   Linux SSH 目标或早于 2012 R2 的 Windows 版本的限制会对您产生影响吗？
*   如果是，您能预见到任何可能阻止您升级这些部署目标或使用替代脚本语言的挑战吗？
*   更新的语言特性，更容易的 NuGet 包引用，以及在移除 Mono 时增加的可靠性证明了这些改变的合理性吗？

您的反馈将帮助我们提供最佳解决方案。

[提供反馈](https://github.com/OctopusDeploy/StepsFeedback/issues/9)

## 结论

总之，从`scriptcs`到`dotnet-script`的迁移将导致以下变化:

*   运行 Mono 的 Linux SSH 目标不赞成使用 C#脚本
*   对于在早于 2012 年 R2 版的版本上运行的 Windows 部署目标，不推荐使用 C#脚本
*   增加对 C# 5 到 C# 8 语言特性的支持
*   脚本中 NuGet 包的直接导入
*   删除了在 Linux 目标上运行 C#脚本的 Mono 要求

感谢您阅读本 RFC。非常感谢您的任何反馈。

愉快的部署！