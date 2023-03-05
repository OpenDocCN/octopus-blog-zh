# docker.io、docker-cd 和 Docker Desktop - Octopus Deploy 之间的区别

> 原文：<https://octopus.com/blog/difference-between-docker-versions>

经过多年的发展，Docker 已经成熟，可以为使用容器的开发人员提供一系列解决方案。这可能会导致一些混乱，因为开发人员需要选择安装哪个版本的 Docker。

在这篇文章中，我将介绍哪些操作系统可以使用哪些选项，并提供如何选择的建议。

## Docker 的多面性

Docker 已经成为一个大的、正在进行的运动的一部分，该运动集中于构建、分发和运行容器化软件。虽然术语“Docker”通常是容器化的同义词，但是理解各种工具和规范协同工作来支持容器化软件是值得的。

支持 Docker 的核心技术由[开放容器倡议](https://opencontainers.org/) (OCI)定义，它用图像规范(image-spec)定义了可分发图像的格式，以及这些图像如何用运行时规范(runtime-spec)运行。

OCI 运行时规范由多个[容器运行时](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction#container_engine)实现，包括 [runc](https://github.com/opencontainers/runc) 、 [crun](https://github.com/containers/crun) 和 [katacontainers](https://github.com/kata-containers/kata-containers) 。容器运行时执行执行容器化流程所需的低级工作。

[容器引擎](https://developers.redhat.com/blog/2018/02/22/container-terminology-practical-introduction#container_engine)如带有[容器的 Docker](https://containerd.io/)和带有 [conmon](https://github.com/containers/conmon) 的 [Podman](https://docs.podman.io/en/latest) 用于通过容器运行时拉取 OCI 映像和启动容器。

开发者和用于构建、分发和运行容器化软件的软件栈之间的接口在技术上是容器引擎，在 Docker 的例子中，容器引擎是`docker`命令行工具。

容器运行时和 Docker 容器引擎的组合被 Docker 网站[称为 Docker 引擎](https://docs.docker.com/engine/)。

现在我们更清楚我们所说的“Docker”是什么意思了，是时候看看开发者在使用[容器化应用](https://octopus.com/blog/get-started-containers)时可用的选项了。

## Linux 用户的选择太多了

容器化技术最初是在 Linux 中开发的，所以 Linux 用户在使用容器化应用程序时有所选择也就不足为奇了。

Docker 并不是 Linux 用户唯一可用的选项。 [Podman](https://docs.podman.io/en/latest/#) 提供了 Docker 的替代品，对于大多数开发人员来说，`podman` CLI 可以配置为`docker` CLI 的别名。

如果你想继续使用 Docker，有两个选择:

*   Debian/Ubuntu 上的`docker.io`或者 Fedora 上的`docker`
*   `docker-ce`

`docker.io`和`docker`包由各自的 Linux 发行版维护。无需添加任何额外的包存储库就可以安装它们。它们是免费和开源的。

`docker-ce`是 Docker 提供的一个包。该软件包可以通过为主要 Linux 发行版提供的第三方软件包存储库获得。像`docker.io`和`docker`软件包一样，`docker-ce`是免费的开源软件。

关于`docker.io` / `docker`和`docker-ce`的底层区别有很多讨论。[stack overflow](https://stackoverflow.com/questions/45023363/what-is-docker-io-in-relation-to-docker-ce-and-docker-ee-now-called-mirantis-k)上的这个问题，有近 9 万的浏览量，列出了每个包的一些利弊。

就我个人而言，我安装`docker-ce`包，因为它通常是最新的。

## Docker 桌面适合所有人

[Docker 桌面](https://docs.docker.com/desktop/)是在 Windows 10 或 11 以及 macOS 操作系统上安装 Docker 引擎的唯一途径。Docker 桌面也适用于 Linux，尽管 Linux 用户可以自由地单独安装 Docker 引擎。

Docker Desktop 是一款商业应用，[要求某些团队付费。](https://docs.docker.com/subscription/#docker-desktop-license-agreement)

## Windows 服务器支持

[Windows server 2016 和 2019 内置支持运行 Docker](https://docs.microsoft.com/en-us/virtualization/windowscontainers/about/#the-microsoft-container-ecosystem) ，但仅支持运行 Windows 容器。

有一些在 Windows 服务器操作系统上运行 Linux 容器的建议。这篇关于服务器故障的文章提供了一个摘要和其他资源的链接。然而，我所看到的信息都没有表明 Windows server 上的 Linux 容器是生产环境支持的解决方案。

## 结论

开发容器化应用程序的开发者首先需要安装一个像 Docker 这样的工具。但是，虽然术语“Docker”是容器的同义词，但有许多选项可供选择，有些根本不需要任何 Docker 专用工具。

在这篇文章中，我们研究了支持容器化应用程序的各种工具和规范，并指出使用容器的开发人员可以选择的方法。

## 使用 Octopus Workflow Builder 了解更多信息

如果您想在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，请尝试使用 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器用一个用 GitHub 操作工作流构建的示例应用程序填充 GitHub 存储库。它用示例部署项目配置了一个托管 Octopus 实例，展示了漏洞扫描和基础设施代码(IaC)等最佳实践。

愉快的部署！