# 八达通部署 3.4 EAP -八达通部署

> 原文：<https://octopus.com/blog/octopus-deploy-3.4-eap>

**Octopus Deploy 3.4 已经发货！阅读[博文](https://octopus.com/blog/octopus-deploy-3.4)和[今天就下载](https://octopus.com/downloads)！**

我们自豪地宣布 Octopus Deploy 3.4 的[早期访问计划(EAP)](http://docs.octopusdeploy.com/display/ODEAP/Octopus+Deploy+EAP) ！这个功能发布是雄心勃勃的，我们想让你早点拿到它，而不是等着发布一个接近成熟的预发布版本。现在绝对是参与的最佳时机，[让我们知道你的想法](#feedback)！

## Octopus Deploy 3.4 出货后会是什么样子？

我们的目标是 Octopus Deploy 3.4 完整版的两个主要特性:

*   多租户部署- [阅读更多信息](https://octopus.com/blog/rfc-multitenancy-take-two)
*   改进的弹性环境和瞬态机器支持- [阅读更多信息](https://octopus.com/blog/rfc-cloud-and-infrastructure-automation-support)然而，我们希望更进一步，甚至做得更好！

## 阿尔法 1 号的盒子里是什么？

我们希望发布整个拼图中有针对性的工作片段，这样您就可以一次从一个概念开始工作，想象每个片段将如何开始组合成一个完整的整体，然后[用您的反馈](#feedback)来回应我们。

### Alpha 1 中的多租户部署

在 Alpha 1 中，您可以:

*   开始使用[租户和租户标签](https://octopus.com/blog/rfc-multitenancy-take-two#tenant-first-class)
*   将租户链接到项目，并为这些项目选择环境。这将允许您配置为每个租户部署哪些项目，以及部署到哪个(哪些)环境中——为复杂场景提供最大的灵活性。
*   为管理租户应用新的安全权限
*   查看为租户创建的新审核事件
*   通过 Octopus API 和客户端库完成所有这些工作！

### Alpha 1 中改进的弹性环境和瞬态机器支持

我们正在测试**机器策略**的概念，它可以控制机器生命周期的某些方面:

*   当计算机联机时(首次联机或从脱机状态恢复时)要做什么
*   自定义运行状况检查脚本(从我们今天的基线/默认操作开始)
*   当计算机在部署开始时离线或在部署过程中离线时该怎么办
*   当机器离线时间过长时该怎么办

在 Alpha 1 中，您可以:

*   在库中构建机器策略
    *   自动删除脱机时间过长的计算机
    *   跳过在部署开始时脱机或在部署期间脱机的计算机
*   对环境中的每台机器应用机器策略
*   为管理机器策略应用新的安全权限
*   查看由计算机策略创建的新审核事件
*   通过 Octopus API 和客户端库完成所有这些工作！

## Octopus Deploy Alpha 1 入门

我们的 EAP 附带 45 天的[企业](https://octopus.com/purchase)试用许可。[下载最新的](http://docs.octopusdeploy.com/display/ODEAP/Octopus+Deploy+EAP) EAP 版本，浏览发行说明。由于还处于开发的早期阶段，我们不支持从 EAP 版本升级。我们建议在试用服务器上安装 EAP 版本。

## 你的反馈真的很重要

请在我们的[论坛](http://help.octopusdeploy.com/discussions/beta-testing-feedback)发表反馈并加入讨论。我们特别希望收到以下方面的反馈:

*   租户到项目/环境的明确映射在多大程度上符合您的情况？
*   机器策略中可用的设置组合是否有助于针对您的情况减少摩擦？

来吧，参与进来，帮助我们构建迄今为止最好的 Octopus 部署。*愉快的部署！*