# Octopus Deploy 收购 Dist - Octopus Deploy

> 原文：<https://octopus.com/blog/octopus-acquires-dist>

今天，我很兴奋地宣布 Octopus Deploy 已经收购了 Dist，一个基于云的容器注册和工件存储服务。

## 为什么 Dist 非常适合

创建 Dist 的目标是为容器和 Java 工件提供高度可靠、快速且易于使用的工件存储库。

我们在去年 10 月认识了 Dist 团队，发现我们有很多共同点。Dist 完全是由悉尼的两位创始人黄云·杨和斯蒂芬·海因斯创建的，他们有很强的技术背景，对这个领域充满热情，我们对他们的成就印象深刻。

在部署应用程序之前，需要将其打包，例如打包成 Zip 文件或 Docker 容器映像，这通常被称为“工件”。每次源代码变更都会产生工件的新版本，由于您可能需要重新部署旧版本，您需要保留不同工件的许多版本，并有效地存储它们，通常存储在称为“工件存储库”的软件中。

从早期开始，Octopus Deploy 就包含了一个内置的工件库。这可以用于推送 NuGet 包、Zip 文件、Java JAR 文件和其他用于部署的工件。客户总是觉得这很方便——少了一项需要管理的服务，而且重要的是，它与我们的[保留政策](https://octopus.com/docs/administration/retention-policies)相集成。今天，虽然我们支持许多类型的工件，但我们目前不支持容器映像，因此客户需要使用第三方解决方案，如 Azure Container Registry 或 AWS Elastic Container Registry，这使得保留策略更加难以管理。

## Dist 将如何帮助我们的客户

Octopus 的使命是加速跨云和内部的可重复、可靠和可跟踪的部署。当我们构建 Octopus Cloud 的功能时，我们希望为客户提供一种方便的方式来使用他们的容器图像，以及潜在的其他工件。Octopus 中集成的低延迟、安全访问和保留策略可帮助您做到这一点，而无需管理额外的服务。Dist 技术和 Dist 团队的专业知识将是我们下一步工作的核心。

## 章鱼是如何成熟的

在过去的几年里，章鱼享受着快速的成长，你可以读到章鱼在 2021 年是如何成长的。在过去，如果我们发现 Octopus Deploy 中缺少的功能，我们的第一反应会是组建一个工程师团队来开发该功能，但随着我们的成长和成熟，我们意识到，从零开始的内部开发并不总是服务客户的最佳方式。

通过与 Dist 团队合作，我们将使软件团队能够更轻松地交付更多功能，减少停机时间，并减少管理工作量。我很高兴地欢迎 Dist 团队加入 Octopus Deploy，迫不及待地想看看我们共同构建的东西！

### 关于 Dist 的炉边聊天

产品总监 Michael Richardson 领导了 Dist 收购过程。他记录了与高级解决方案架构师 Derek Campbell 的一次聊天，内容是关于 Dist technology 将如何使用 Octopus 改进部署。

[https://www.youtube.com/embed/0CAFUTWW6-k](https://www.youtube.com/embed/0CAFUTWW6-k)

VIDEO

愉快的部署！