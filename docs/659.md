# 为你八达通系统进行安全检查

> 原文：<https://octopus.com/blog/security-checkup>

你会照顾自己的章鱼吗？我们也是。我们仍然有自己的 Octopus 服务器，作为部署我们构建的大多数东西的宠物。就像我们一样，大多数人可能都有自己的宠物章鱼——但你的章鱼有多安全呢？

我们最近发布了章鱼云 alpha，在那里我们学到了，也重新学到了很多关于如何让章鱼安全的经验。在这篇文章中，我将向你展示如何给你的章鱼做安全检查。

*不想再照顾自己的章鱼了？也许你应该今天就报名参加[章鱼云](https://octopus.com/cloud)？*

想要完整的图片吗？这里是我们关于强化章鱼的深度指南。

## 安全地暴露你的八达通服务器

为了让 Octopus Server 做有用的事情，您需要向您的用户、您的基础设施以及可能的外部服务公开它。如果你将你的 Octopus 服务器暴露给公共互联网或第三方，你绝对应该使用 SSL 上的 HTTPS。有了对[的内置支持，让我们加密](https://octopus.com/docs/administration/security/exposing-octopus/lets-encrypt-integration)，就没有借口在 HTTP 上公开你的 Octopus 服务器了。

了解如何安全地暴露你的八达通服务器。

另外，你知道 Octopus 与代理服务器 T1 一起工作吗？

## 安全地在你的八达通服务器上工作

您的部署过程通常会处理包并执行脚本。这些包经常被推送到一个触手或 SSH 部署目标，您的脚本将在这些机器上执行。然而，许多部署并不需要像对云服务或类似的部署那样的触手或 SSH 目标。在这种情况下，如果您不得不设置一个 Tentacle 或 SSH 目标，只是为了将一个包推送到一个 API 或运行一个脚本，但您并不关心该脚本在哪里运行，这将是令人恼火的。

在 Octopus `3.0`中，我们引入了 worker 的概念，它可以处理包和执行脚本，而不需要安装和配置触手或 SSH 目标。默认情况下，使用[内置工作器](https://octopus.com/docs/administration/workers/built-in-worker)的 Octopus 服务器运行在与 Octopus 服务器相同的安全上下文中。当你开始跑步时，这非常方便，但这不一定是长期坚持跑步的最佳方式。

我们建议将每台 Octopus 服务器配置为使用内置工作器作为不同的用户，或者使用[外部工作器](https://octopus.com/docs/administration/workers/external-workers)。

了解[工人](https://octopus.com/docs/administration/workers)。

## 强化您的主机操作系统

你的八达通服务器的主机操作系统有多安全？如果你不确定，[这里有一些提示](https://octopus.com/docs/administration/security/hardening-octopus#harden-your-host-operating-system)！

## 强化您的网络

你允许自由进出你的八达通服务器吗？Octopus 实际上使用了少量定义明确的网络协议。我们还提供了一些提示，可以防止攻击者使用您的 Octopus 服务器作为进入您网络的媒介。

了解[强化你的网络](https://octopus.com/docs/administration/security/hardening-octopus#harden-your-network)。

## 提升

你运行的是最新版本的八达通服务器吗？一般来说，最新版本的八达通服务器将是最安全的。你应该考虑一个保持你的 Octopus 服务器更新的策略。我们遵循[负责任的披露政策](https://octopus.com/docs/administration/security#disclosure-policy)，因此您有可能知道任何影响您的八达通服务器的安全性和完整性的已知问题。

## 结论

Octopus 是以安全为首要考虑因素的——但是[你仍然需要做一些工作来保证你的 Octopus 的安全。这篇博文有一些提示和技巧。如果你想给你的章鱼做一个全面的检查，使用我们的](https://octopus.com/docs/administration/security/hardening-octopus)[深度指南来强化章鱼](https://octopus.com/docs/administration/security/hardening-octopus)。

如果你不想照顾自己的章鱼，也许你应该今天就报名[章鱼云](https://octopus.com/cloud)？