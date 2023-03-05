# 章鱼 TL；灾难恢复-性能改进- Octopus 部署

> 原文：<https://octopus.com/blog/tldr-2017-04-12-performance>

每周三早上，我们会召开一个简短的全公司会议，我们称之为“TL；“博士。这是一个公开邀请，任何团队成员都可以展示他们认为相关或有趣的东西，以及他们希望公司其他人知道的东西。它们涵盖了各种各样的主题，从特性设计到从支持事件中吸取的经验教训，再到开发人员在构建软件时可能会用到的模式。

作为一项实验，我们将把这些演示文稿每周公之于众——至少是那些不太机密的部分😉我希望它能让你对 Octopus 的内部有一点“幕后”的了解。

今天的 TL；DR 展示了两个关于性能的演示。Octopus 3.12 具有一些非常显著的性能改进，这要感谢一些报告问题的客户，他们能够提供跟踪和其他信息来帮助我们重现问题。

首先，Rob Erez 分享了一些即将到来的与浏览器缓存相关的改进，并减少了我们的 web 请求总数。然后，迈克尔·诺南分享了他最近使用 [JetBrains dotMemory](https://www.jetbrains.com/dotmemory/) 来[诊断一些内存问题](https://github.com/OctopusDeploy/Issues/issues/3398)的过程。我希望你喜欢！

[https://www.youtube.com/embed/eByv1uuum88](https://www.youtube.com/embed/eByv1uuum88)

VIDEO

我很想知道这是否是你感兴趣的事情，以及是否有你想看到我们讨论的话题。请在下面的评论中留言，别忘了订阅 YouTube 频道！