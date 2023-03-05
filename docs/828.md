# 在 Windows Server 2008 上终止对 Octopus 服务器的支持。介绍 Linux 上的 Octopus 服务器！-章鱼部署

> 原文：<https://octopus.com/blog/windows-server-2008-eol-hello-linux>

章鱼服务器 2019.3 LTS 将是你可以在 Windows Server 2008 上托管的章鱼服务器的最终版本。

这个决定为真正的跨平台 Octopus 服务器铺平了道路。在不久的将来，你将能够在任何现代的 Windows 或 Linux 操作系统上托管 Octopus 服务器，或者在任一平台上托管一个容器。为此，我们需要将托管 Octopus 服务器的最低要求提高到 **Windows Server 2008 R2** 。

同样值得注意的是，微软将于 2020 年 1 月结束对 Windows Server 2008 和 2008 R2 的扩展支持，谢天谢地，你可以[执行就地升级](https://docs.microsoft.com/en-us/windows-server/get-started/installation-and-upgrade)。如果 Octopus 服务器是您业务中的关键资源，您真的应该考虑将其托管在现代操作系统上，以提高安全性和性能。

这篇博文的其余部分应该回答最常见的问题。一如既往，如果您有任何问题或疑虑，请联系我们的支持团队！愉快的部署！

## 问题:我会受到影响吗？

只有在以下情况下，您才会受到影响:

1.  您在 Windows Server 2008 上托管 Octopus 服务器，并且
2.  你想升级 Octopus 服务器超过`2019.3 LTS`。

别担心，八达通服务器安装程序会防止你意外升级。如果你想升级 Octopus 服务器，你需要升级你的主机操作系统。

## 问:我们还能在 Windows Server 2008 R2 上托管 Octopus 服务器吗？

是的，Windows Server 2008 R2 仍将是 Octopus 服务器支持的主机。

尽管微软将于 2020 年 1 月终止对 Windows Server 2008 R2 版的扩展支持，但我们现在没有排除 Windows Server 2008 R2 版的实际理由。

## 问:我的部署会受到影响吗？

不，您的部署不会因升级 Octopus 服务器超过`2019.3 LTS`而受到影响。这只影响八达通服务器本身的托管。我们仍然高度向后兼容您在 Octopus 中的部署目标，甚至可以追溯到`Windows Server 2003`。

## 问:如何升级我的主机操作系统？

您可以将 Windows Server 2008 操作系统就地升级到更现代的操作系统。了解如何升级 Windows 服务器。

或者，你可以将你的八达通服务器转移到另一台已经运行更现代的操作系统的主机上。了解[移动你的章鱼服务器](https://octopus.com/docs/administration/managing-infrastructure/moving-your-octopus)。

## 问题:你们会支持八达通服务器 2019.3 LTS 和 Windows Server 2008 多久？

我们将根据我们的[长期支持计划](https://octopus.com/docs/administration/upgrading/long-term-support) **继续支持 Octopus 服务器`2019.3 LTS`，直到 2019 年 10 月**。要升级 Octopus 服务器超过`2019.3 LTS`，您需要将您的主机操作系统升级到 **Windows Server 2008 R2 或更新的**。

## 问:什么时候可以在 Linux 上托管 Octopus 服务器？

在不久的将来。我们内部已经在这么做了。Octopus Server `2019.5`，我们目前在快车道上发布的版本，将开始引入跨平台功能。在此之后，我们将关注 Linux 上 Octopus 服务器的稳定性和性能，随后是运行在 Linux 上的 Octopus 服务器的完全支持版本。