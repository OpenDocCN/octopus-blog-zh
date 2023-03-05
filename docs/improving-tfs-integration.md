# 用 TFS 配章鱼？我们想听听你的意见！-章鱼部署

> 原文：<https://octopus.com/blog/improving-tfs-integration>

一段时间以来，我们为 JetBrains TeamCity 开发了一个[插件，使得在构建完成时部署发布变得非常容易。不幸的是，集成 Octopus Deploy 和 Microsoft Team Foundation Server 的故事从来没有这么顺利过。作为一项规则，使 TFS 和八达通一起工作通常涉及:](http://docs.octopusdeploy.com/display/OD/TeamCity)

*   自定义 MSBuild 脚本
*   团队构建工作流更改

我真的很想改善这种体验；事实上，我希望 Octopus 成为 TFS 客户最简单、最有吸引力的自动化部署体验。

为了做到这一点，我们计划与 TFS 进行更深入的整合，要么作为 Octopus 中的一项功能，要么作为连接两者的外部服务。目标是在团队构建完成后，用 Octopus 自动创建和部署发布，*不需要任何工作流更改。*

例如:

*   团队构建完成时自动创建发布
*   在发行说明中，包含与该版本关联的所有工作项的链接
*   将版本部署到冒烟测试环境中
*   当团队构建的构建质量发生变化时，将其部署到另一个环境中

我们仍处于集思广益阶段，这就是我们需要你帮助的地方。如果您使用(或计划使用)TFS 和章鱼，我们希望您[填写这份 30 秒的调查](https://docs.google.com/forms/d/1Yl1ALfkXENxk6bkzxuUyZVCEsnKmIX_eYz_z8sdnalY/viewform)。它只有 5 个问题，我们不要求您在 10 分的范围内给任何问题打分！