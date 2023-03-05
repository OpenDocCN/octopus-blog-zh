# Octopus 工作流生成器反馈- Octopus 部署

> 原文：<https://octopus.com/blog/octopus-workflow-builder-feedback>

现代开发运维团队对他们的持续交付工作流有什么要求？持续集成、云部署、特性分支、测试、软件材料清单(SBOMs)和依赖性漏洞扫描只是高绩效团队快速交付和维护高质量软件所需的一些特性。

但是你如何*实际上*实现这些过程呢？多年来，我们在这个博客上分享了很多意见，以帮助团队从 Octopus 中获得最大收益，但感觉我们经常在讲述故事的结尾。我们希望“使复杂的部署变得简单”，但是我们经常跳过设置甚至是最简单的真实样本环境的单调工作。作为读者，您需要编写自己的示例应用程序，实例化自己的云基础设施，填充自己的 Octopus 配置，并在遵循我们最新的操作指南之前复制/粘贴我们的示例。

我们希望您能够体验到现代连续交付工作流程对您实际使用的平台的威力和乐趣，而无需花费数天时间来设置您的工具。这就是我们构建 [Octopus 工作流构建器](https://octopusworkflowbuilder.octopus.com/#/)的原因，我们希望您对这个早期版本有所反馈。

## 什么是 Octopus Workflow Builder

[Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 提供了一个简单的向导，您可以在其中选择想要部署到的平台(本版本支持 EKS、ECS 和 Lambdas)，输入想要用示例部署项目填充的 Octopus cloud 实例，输入您的 AWS 密钥，并授权应用程序代表您创建 GitHub 存储库。

然后，[工作流构建器](https://octopusworkflowbuilder.octopus.com/#/)将使用以下内容填充 GitHub 存储库:

*   示例 web 应用程序
*   Terraform 配置文件，用于创建 ECR 存储库并使用 [Octopus Terraform 提供者](https://registry.terraform.io/providers/OctopusDeployLabs/octopusdeploy/latest/docs)填充 Octopus 空间
*   GitHub Actions 工作流编译、测试和发布示例应用程序，并应用 Terraform 配置

这进而用以下内容填充您的 Octopus 实例:

*   环境、帐户、生命周期、源和目标
*   一个基础架构部署项目，创建一个 EKS 集群和 ECS 集群，或者创建一个 API 网关来公开 Lambdas
*   一个后端部署项目，它部署一个 RESTful API，对其进行冒烟测试，并执行与 Postman 的集成测试
*   一个前端部署项目，它部署一个静态 web 应用程序，对其进行冒烟测试，并使用 Cypress 执行端到端测试
*   一种环境，在该环境中，根据关联应用程序的依赖项生成的 SBOM 包执行安全扫描

最终结果是一个自以为是的 CI/CD 工作流，用于部署、测试和维护真正的基于云的应用程序。最棒的是，整个工作流程是在您自己的 GitHub 存储库和 Octopus 实例中配置的，因此您可以完全控制以您想要的方式定制流程。

以下视频重点介绍了由工作流构建器创建的示例部署项目的功能:

[https://www.youtube.com/embed/wABZvJPVCMg](https://www.youtube.com/embed/wABZvJPVCMg)

VIDEO

[https://www.youtube.com/embed/vcHdGRS-xzU](https://www.youtube.com/embed/vcHdGRS-xzU)

VIDEO

[https://www.youtube.com/embed/sex-QLKA5xE](https://www.youtube.com/embed/sex-QLKA5xE)

VIDEO

[https://www.youtube.com/embed/Wo4JY8fV_WM](https://www.youtube.com/embed/Wo4JY8fV_WM)

VIDEO

## 我们需要您的反馈

我们希望得到您对这个早期版本的 [Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 的反馈，以帮助我们消除任何错误，并了解该工具是否对您有用。

我们有一个 [GitHub 问题](https://github.com/OctopusSamples/content-team-apps/issues/13)，你可以在这里提交任何反馈。

## 访问源代码

如果你对[工作流构建器](https://octopusworkflowbuilder.octopus.com/#/)的源代码感兴趣，我们在 [GitHub](https://github.com/OctopusSamples/content-team-apps) 上发布了它。

## 结论

我们希望您喜欢 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) ，并期待您的任何反馈，以帮助我们使它成为最好的工具。

[提供反馈](https://github.com/OctopusSamples/content-team-apps/issues/13)

愉快的部署！