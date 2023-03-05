# Octopus 部署中的停靠引擎- Octopus 部署

> 原文：<https://octopus.com/blog/docker-engine-in-octopus-deploy>

## Octopus Deploy 中现在提供了基本的 Docker 引擎命令

在我们谨慎地提出我们对 Octopus 如何与 Docker 整合的想法后，来自社区的普遍反馈虽然积极，但似乎有点不确定。这可能部分是因为 Docker 对我们大多数人来说是一项相当新的技术。净客户群，也是由于一个*的案例，看到就知道是否管用。*由于这个原因，我们发布了新的 Docker 功能作为早期访问功能，但捆绑在最新的 Octopus Deploy 3.5.1 版本中。![](img/a9ab5829d59e38c56cffc12467dbd449.png)

## 在你的八达通服务器上启用 Docker

因为我们想清楚地表明这些功能仍在开发中，所以我们默认隐藏了它们，它们只能通过切换 Docker 功能标志来启用。一旦这个特性被打开，Docker 步骤就可以在项目中使用，Docker 注册中心也可以作为外部提要来创建。一旦特性完全稳定下来，我们希望它们在默认情况下是可见的，并且切换被移除。![Enable Docker Feature Flag](img/6caa92362c1fdc92133a4106ffad33ce.png)

值得注意的是，这是因为在下一个版本中，Docker 特性集可能会引入一些突破性的变化。我们希望用户和早期采用者能够接触到这个版本，一起和章鱼& Docker 一起玩，看看什么有用，什么没用。运行容器如何适应您的发布周期？

## 码头进料

3.5.1 支持新的提要类型，允许您在发布创建时从 Docker 注册表中选择 Docker 映像，就像您习惯于使用 NuGet 提要一样。我们现在有一些其他有趣的想法，提要不一定需要成为 NuGet 库...所以看好这个空间；)

了解更多关于 [Docker 注册表提要](http://docs.octopusdeploy.com/display/OD/Docker+Registries+as+Feeds)的信息。

## 码头台阶

到目前为止，我们正在为`docker run`、`docker stop`和`docker network create`提供一流的支持。我们的目标是从支持 Docker 生态系统的基本构建模块开始，然后在它们之上构建并覆盖顶层抽象，如 Docker Compose 和 Docker Swarm。最终，我们还计划覆盖与 Kubernetes、Apache Mesos 以及 AWS 和 Azure Container Services 等云选项的集成。

查看我们的指南，开始一个基于 Docker 的基本项目。![Docker Step](img/1ec9b7efcb3293549e8a89e3125fe48b.png)

## 限制

在此版本中，Docker 步骤有一些已知的限制；

*   Docker Hub 的 API 和官方注册 API 规范之间有一些细微的差别。此外，由于 Docker Hub 与普通注册表相比暴露了关于图像的额外信息(例如，它们是[官方图像](https://docs.docker.com/docker-hub/official_repos/)？它们是公开的还是私人的？)信息的可访问性各不相同。虽然公共的、官方的存储库在这个版本中运行良好，但是非官方图片的标签不会返回，私有的存储库也不会暴露。建立自己的存储库非常简单(当然是使用容器！)目前，如果试图从非官方 Docker Hub 映像运行容器，我们建议使用私有存储库。

*   Windows 上的 Docker 仍然有一些问题需要解决，特别是在网络和容量方面，所以目前我们建议在 Linux 目标上进行尝试。这是一个我们确实希望在未来版本中得到同等功能的领域，但是这很大程度上依赖于 Docker 和微软来实现。

## 你觉得怎么样？

正如已经提到的，Docker 到 Octopus Deploy 的集成将经历几个阶段，在每一步，我们都希望确保我们正在构建您完成工作所需的东西。这个第一阶段有望提供这个集成看起来像什么的指示，即使有一些改进要做。今天下载 Octopus Deploy 的最新版本,让我们知道你的想法！