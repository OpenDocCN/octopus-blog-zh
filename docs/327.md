# 提高 PowerShell 兼容性- Octopus Deploy

> 原文：<https://octopus.com/blog/improving-powershell>

Octopus 允许在部署期间执行 [PowerShell 脚本](http://octopusdeploy.com/documentation/features/powershell)。PowerShell 非常强大，是实现部署自动化的一个很好的工具，所以它是我们非常依赖的一个特性。

为了调用 PowerShell 脚本，我们目前使用 [Runspace](http://msdn.microsoft.com/en-us/library/system.management.automation.runspaces.runspace(v=vs.85).aspx) 类在 Octopus 触手进程中托管 PowerShell。我们将 PowerShell 脚本从一个文件读入一个字符串并执行它，就像您在 PowerShell 控制台中输入它一样。

## 虫子，哦，虫子...

通过在我们自己的进程中托管 PowerShell，与直接使用 PowerShell.exe 运行脚本相比有一些不同。自从 Octopus 首次发布以来，我们一直在不断提高兼容性水平，我们已经进行了大量的集成测试，这些测试是在不同的 Windows 配置上运行的(2003、2008，包括 R2 和 2012、x86 和 x64)。大多数时候，它只是工作。除了它不在的时候！

几乎每周我们都会收到一份类似于以下内容的错误报告:

> 我有一个 PowerShell 脚本，当我从 PowerShell 运行它时运行良好，但是当我在触手下运行它时，我得到...

示例:

虽然有很多关于“如何托管 PowerShell 是一个. NET 应用程序”的例子，但我从未见过关于“如何托管一个 100%兼容 PowerShell.exe 的‘一切正常工作’！”的权威指南. NET 应用程序中的主机”，这两者之间有很大的区别。有很多开源应用程序使用与我们非常相似的托管代码，但是它们似乎都有这些问题。

## 过程。开始救援！

为了避免看到更多的错误报告，从 Octopus 1.4 开始，我们将切换到一个新的 PowerShell 调用模型。我们要打电话给 PowerShell.exe，告诉他结果。

如果你有兴趣大致了解我们将如何调用 PowerShell，[查看这个要点](https://gist.github.com/PaulStovell/5037973)。它还有一个好处，就是比我们目前的模型简单一百倍。

这带来的一个问题是向后兼容性和对现有脚本的支持，现有脚本可能会在新模型下崩溃。

## 多种 PowerShell 模式？

最初，我计划支持多种 PowerShell 调用模式，这样就可以选择是否使用内存(当前)模式还是 PowerShell(新)模式。用户可以在设置包或脚本步骤时进行选择。

但是这太复杂了。谁想在调用 PowerShell 脚本的四种方法中进行选择？我们想要处理四种不同 PowerShell 调用 mdoels 的错误报告吗？此外，就 Octopus 而言，在进程中托管 PowerShell 没有任何优势。它实际上更慢，因为我们必须创建和拆除 AppDomain。除了遗留支持之外，我根本找不到任何保留现有模型的理由。

退一步说，鼓励人们编写只在触手下运行时有效，而在 PowerShell.exe 下运行时无效的脚本是个好主意吗？如果“锁定”是我们的策略那么也许。但我们在 NuGet 和 PowerShell 等标准平台上构建 Octopus 正是为了避免这种情况。

## 这意味着什么

目前的计划是:

1.  从 1.4 开始，使新模型成为调用 PowerShell 脚本的默认模型(因此它应该“适合”所有人)
2.  通过设置一个特殊变量来请求“遗留”PowerShell 模式，使旧模型可用，然后在未来的版本(可能是 1.5 或 1.6)中删除该选项

我不认为在我们的“遗留”(托管)模式下会有很多人依赖某个行为的情况，但如果有，我们会在一两个版本中支持他们。否则，人们应该准备转向新的 PowerShell 模式。在 99%的情况下，它会工作。在那 1%没有的地方，无论如何都应该修复。