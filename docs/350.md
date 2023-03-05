# 构建服务器和持续集成简介- Octopus Deploy

> 原文：<https://octopus.com/blog/introduction-to-build-servers>

当您开发和部署软件时，首先要解决的事情之一是如何将您的代码和工作应用程序部署到人们可以与您的软件交互的生产环境中。

大多数开发团队理解版本控制对协调代码提交的重要性，并构建服务器来编译和打包他们的软件，但是持续集成(CI)是一个大话题。

在我们关于[持续集成](https://octopus.com/blog/tag/CI%20Series)的博客系列中，我们将详细介绍持续集成，以及两个最流行的构建服务器 [Jenkins](https://www.jenkins.io/) 和 [GitHub Actions](https://github.com/features/actions) 如何帮助您的 CI 流程。

## 为什么构建服务器很重要

构建服务器有 3 个主要目的:

*   一天多次从您的存储库中编译提交的代码
*   运行自动测试来验证代码
*   创建可部署的包并交给部署工具，比如 Octopus Deploy

没有构建服务器，您会被复杂的手动过程和它们引入的不必要的时间限制所拖累。例如，如果没有构建服务器:

*   您的团队可能需要在每天的截止日期之前或者在变更窗口期间提交代码
*   截止日期过后，没有人可以再次提交，直到有人手动创建并测试一个构建
*   如果代码有问题，截止日期和手动过程会进一步延迟修复

没有构建服务器，团队就要克服自动化所消除的不必要的障碍。构建服务器将全天为您重复这些任务，没有那些人为的延迟。

但是 CI 不仅仅意味着花费在手工任务上的时间更少或者任意截止日期的死亡。通过每天多次自动采取这些步骤，您可以更快地解决问题，并且您的结果变得更加可预测。构建服务器最终会帮助您更加自信地通过管道进行部署。

## 詹金斯是什么？

[Jenkins](https://www.jenkins.io/) 是市场上最受欢迎的 CI 平台。开源且免费使用，您可以在大多数操作系统上独立运行 Jenkins 来自动构建和测试您的代码。

詹金斯最大的好处之一是它的灵活性。这是一个可扩展的平台，意味着您可以随着您的团队或项目需求的增长而扩展其功能。得益于其庞大的社区，[有超过 1800 个插件](https://plugins.jenkins.io/)，这使得它很容易与无数的行业工具集成。

这意味着 Jenkins 足够灵活，可以满足您的 CI 需求，您也可以为其他自动化目的定制它。

## 什么是 GitHub Actions？

[GitHub Actions](https://github.com/features/actions) 是较新的 CI 平台之一。它通过使用存储库事件来触发虚拟“运行者”上的自动化工作流，从而消除了对单独构建服务器的需求。

如果你使用 GitHub 作为你的代码仓库，好消息是你已经有了访问权限——GitHub Actions 包含在你现有的仓库中。

关键在哪里？虽然公共存储库可以免费使用 GitHub 操作，但对其他人来说，它是按需付费的，按工作流运行的时间按分钟计费。不过，所有用户每月都有免费的通话时间，只有超出套餐允许的时间，你才需要付费。

像 Jenkins 一样， [GitHub 也有一个 actions marketplace](https://github.com/marketplace) ，充满了社区创建的应用和工作流，以帮助持续集成等等。

作为 GitHub 的官方技术合作伙伴，Octopus Deploy 在 GitHub 市场上有一系列[验证的 Octopus 操作](https://github.com/marketplace?query=octopus&type=actions&verification=verified_creator)来帮助您的部署。

还可以了解更多关于[用 GitHub 构建，用 Octopus](https://octopus.com/github) 部署。

## 下一步是什么？

在我们的 [CI 系列](https://octopus.com/blog/tag/CI%20Series)中，我们分享了 Jenkins 和 GitHub 行动指南，以及更多内容。

如果你还没有使用 Octopus Deploy，你可以[注册免费试用](https://octopus.com/start)。

您还可以查看一些关于构建服务器和 CI 的其他帖子:

愉快的部署！