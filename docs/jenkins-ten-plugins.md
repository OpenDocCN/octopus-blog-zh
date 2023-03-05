# 我们最喜欢的 10 个 Jenkins 插件- Octopus Deploy

> 原文：<https://octopus.com/blog/jenkins-ten-plugins>

作为一个开源的持续集成(CI)平台，我们最喜欢 Jenkins 的一点是它强大的社区。这一点在 Jenkins 插件索引中表现得最为明显。

索引中有超过 1800 个用户创建的插件，允许您扩展 Jenkins 的功能并更改您的实例以满足您团队的需求。

这里有 10 个我们最喜欢的 Jenkins 插件，它们能给你的管道带来什么，以及如何安装它们。

## 1:蓝色海洋

詹金斯自己的[蓝海插件](https://plugins.jenkins.io/blueocean/)用现代的外观和感觉更新了用户界面。

简单、直观的设计使创建管道、读取流程状态和发现管道问题变得更加容易。此外，它允许您创建自己的仪表板，因此您只能看到您需要的东西。

## 2:简单的主题

使用简单主题插件，你可以使用 CSS 和 JavaScript 改变 Jenkins 的外观和感觉。

你也可以通过[插件的索引页面](https://plugins.jenkins.io/simple-theme-plugin/#plugin-content-themes)在 GitHub 上找到现成的主题。

## 3:性能发布者

如果你不确定如何阅读数据，数据是没有用的，这就是为什么 [Performance Publisher](https://plugins.jenkins.io/perfpublisher/) 插件如此有用。

Performance Publisher 帮助您发现测试结果中的趋势。它读取由您的测试工具创建的 XML 文件，并将它们转换成易于阅读的图表和统计数据。

## 4: GitHub

GitHub 插件将詹金斯连接到 GitHub。这允许交叉功能，例如:

*   两个服务之间的超链接
*   构建状态报告
*   挂钩触发器
*   手动和自动模式

参见[插件的 Jenkins 索引页面](https://plugins.jenkins.io/github/)了解更多信息，包括示例管道。

## 5: Kubernetes

当您在 Kubernetes 集群中运行 Jenkins 时，Kubernetes 插件有助于自动扩展。

这个插件为每个通过变量自动连接的动态代理创建一个 pod。集群中的一个容器充当 Jenkins 代理，允许您在其他容器上运行您需要的任何作业或进程。

## 6:仪表板视图

[仪表板视图插件](https://plugins.jenkins.io/dashboard-view/)让你在 Jenkins 中创建新的视图和门户。您可以从一系列 portlets 构建您的仪表板，以可理解的方式帮助您跟踪作业、测试等。

## 7: Maven 集成

作为一个流行的 Java 项目构建自动化工具， [Maven 插件](https://plugins.jenkins.io/maven-plugin/)为那些在管道中运行 Apache Maven 的人提供了新的特性。

该插件扩展了 Jenkins 的兼容性，以更好地支持:

*   成功构建后触发作业和部署
*   其他[报表插件的自动设置](https://plugins.jenkins.io/ui/search?sort=relevance&categories=&labels=report&view=Tiles&page=1&query=)

## 8:文件夹

[文件夹插件](https://plugins.jenkins.io/cloudbees-folder/)非常有用，詹金斯在安装时推荐了它。这个简单的插件通过让你在嵌套文件夹中分组作业来帮助你排序你的实例。

您可以根据每个文件夹的用途为其提供自己的专用视图，帮助清理 UI 中的混乱。

## 9:引导式多测试结果报告

[bootstrapped-multi-test-results-report](https://plugins.jenkins.io/bootstraped-multi-test-results-report/)是一个可视化工具，可以让你制作测试结果的深度 HTML 报告。该插件支持 Cucumber、Junit、RSpec 和 TestNG 等。

查看插件网站上的[示例报告。](https://web-innovate.github.io/cucumber-reports/featuresOverview.html)

## 10:管道实用程序步骤

[管道实用程序步骤插件](https://plugins.jenkins.io/pipeline-utility-steps/)提供了许多额外的步骤来帮助您处理 Jenkins 管道中的小事情。

使用此插件，您可以:

*   查找、管理和创建文件
*   读写常见的配置文件，包括 CSV、JSON 和 YAML
*   压缩和解压缩文件
*   读写 Maven 模型数据结构
*   比较两个版本号

查看插件的 GitHub 页面，了解[完整的步骤列表和使用它们所需的语法](https://github.com/jenkinsci/pipeline-utility-steps-plugin/blob/master/docs/STEPS.md)。

## 额外推荐:章鱼部署

如果我们不提到我们自己的 [Octopus 部署插件](https://plugins.jenkins.io/octopusdeploy/)，它将 Octopus 连接到您的 Jenkins 管道，那就不对了。

一旦 Jenkins 完成了代码的编译、测试和打包，我们的插件就可以:

*   构建完成时自动触发 Octopus 中的部署
*   如果在 Octopus 中的部署失败，则在 Jenkins 中的构建失败

请参阅我们的文档，了解更多关于将 Jenkins 与 Octopus 一起使用的信息。此外，请查看我们其他一些与詹金斯相关的博客:

如果你还没有使用 Octopus Deploy，你可以[注册免费试用](https://octopus.com/start)。

## 如何安装 Jenkins 插件

您可以使用`install-plugin`命令通过 Jenkins web UI 或 [Jenkins CLI(命令行界面)](https://www.jenkins.io/doc/book/managing/cli/)安装插件。

要安装 Jenkins 插件，您必须:

*   配置 Jenkins 控制器以允许从更新中心下载元数据(在 Jenkins 设置期间配置)
*   作为 Jenkins 实例的管理员

在你安装一个插件之前，通读它的文档。Jenkins 将在安装过程中安装所有依赖插件，但请务必注意任何其他先决条件或额外步骤。

有关管理和安装插件的更多详细信息，请参见 Jenkins 的文档，包括:

### Jenkins web 用户界面

要从 Jenkins web UI 搜索和安装插件:

1.  从左侧菜单中点击**管理詹金斯**并选择**管理插件**。
2.  单击可用的**选项卡，并使用搜索框从更新中心查找插件。**
3.  勾选所需插件左侧的方框，然后点击**不重启安装**或**立即下载并重启后安装**。您可以安装大多数插件，无需重启。

### 詹金斯 CLI

使用以下命令行通过 Jenkins CLI 安装插件。将`[SOURCE]`替换为本地文件、URL 或更新中心，以便从以下位置安装:

```
java -jar jenkins-cli.jar -s http://localhost:8080/ install-plugin [SOURCE] 
```

您也可以在命令行末尾使用这些修饰符:

*   `-deploy`:不重启安装插件
*   `-name VAL`:添加一个短名称(默认情况下，Jenkins 使用源代码中出现的插件名称)
*   `-restart`:插件安装成功后重启 Jenkins

## 下一步是什么？

这些是我们最喜欢的插件，可以帮助你的詹金斯管道，但我们只是触及了表面。在 [Jenkins Index](https://plugins.jenkins.io/) 上可以找到更多可以进一步帮助你进行 CI 工作的信息。

尝试我们免费的 Jenkins 管道生成器工具来用 Groovy 语法创建一个管道文件。这是您启动管道项目所需的一切。

## 观看我们的詹金斯管道网络研讨会

[https://www.youtube.com/embed/D_7AHTML_xw](https://www.youtube.com/embed/D_7AHTML_xw)

VIDEO

我们定期举办网络研讨会。请参见[网络研讨会第](https://octopus.com/events)页，了解关于即将举办的活动和实时流媒体录制的详细信息。

阅读我们的[持续集成系列](https://octopus.com/blog/tag/CI%20Series)的其余部分。

愉快的部署！