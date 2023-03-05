# 触手。NET 版本更改- Octopus 部署

> 原文：<https://octopus.com/blog/tentacle-dotnet-version-change>

与。NET Framework 4.5.2 将于 2022 年 4 月 26 日结束支持。NET Core 3.1 将于 2022 年 12 月 13 日结束支持，我们将把触手迁移到。仅适用于 Windows installer 的. NET Framework 4.8，以及。NET 6 来实现其他功能，包括 Windows Docker 映像。

在这篇文章中，我解释了我们的决定，并回答了一些关于这些版本变化的问题。

## 为什么我们现在要移动触手？

触手已经展开。NET Framework 4.5.2 和。网芯 3.1 一段时间了。我们转移这些版本是因为。NET Framework 4.5.2 于 2022 年 4 月 26 日结束支持。NET Core 3.1 将于 2022 年 12 月 13 日结束支持。在此之后，我们将错过安全补丁和库更新。

为了继续为我们的客户提供安全的软件，我们必须与我们的底层开发框架的支持时间表保持一致。

## 触手前进的计划是什么？

从版本 6.3 开始，触手将需要。用于 Windows installer 的. NET Framework 4.8。NET 6 用于其他一切。仍有许多部署目标使用不同版本的 Windows 7 和 8(包括 Windows 7 SP1 版和 Windows Server 2008 SP2 版)，它们需要一个受支持的。短期到中期的网络版本。

在不久的将来，我们将停止对的支持。NET 框架，让一切都在上面运行。NET 6。

## 你可能有的问题

### 我需要做些什么吗？网络改版？

| 操作系统（Operating System） | 行动 |
| --- | --- |
| Windows(Windows Server 2022 之前) | 安装。NET Framework 4.8 运行时 |
| Windows (Windows Server 2022 或更高版本) | 没有任何东西 |
| Linux/MacOs | 确保操作系统与兼容。网络 6 |

当触手 6.3 可用时，您可以使用我们支持的任何方法升级您现有的触手。

了解有关兼容的更多信息。请访问以下 Microsoft 文档页面:

### 我的部署会受到影响吗？

只要您的触手操作系统满足运行时需求，您的部署就不会受到这种移动的影响。通过包含。NET 框架 4.8，我们仍然保持向后兼容，早在 Windows 7 SP1。

### 我应该升级 Windows 7 SP1 版和 Windows Server 2008 SP2 版上的部署目标吗？

是的，我们鼓励您升级仍在 Windows 7 SP1 版或 Windows Server 2008 SP2 版上的产品。

我们建议升级到至少支持的版本。NET Framework 4.8，以及理想情况下与。NET 6 及以上。

### 如果我不能升级我的部署目标怎么办？

如果您无法升级到受支持的。NET 版本，你需要锁定你的触手版本，避免它自动升级。锁定确保你的触须保持功能。本次升级前的触手最新版本是 6.2.277。

要了解更多关于锁定你的触手版本，请阅读[我们关于触手版本的帖子](https://octopus.com/blog/tentacle-versioning#lock-on-the-tentacle)。

### Windows Installer 何时会升级到。NET 6？

我们计划将触手移动到。在 2023 年初至 2023 年中期，Windows installer 将使用. NET 6。这使您有时间将部署目标升级到支持的版本。NET 6。

## 结论

我们希望这篇文章能解释为什么我们要把触手移动到。NET Framework 4.8 和。NET 6。

如果您有任何问题或顾虑，请联系我们的[支持团队](mailto:support@octopus.com)。

愉快的部署！