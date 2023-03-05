# 在本地测试 Kubernetes-Octopus 部署

> 原文：<https://octopus.com/blog/testing-kubernetes-locally>

Kubernetes 是一个复杂的平台，通常部署在许多服务器上以实现高可用性。然而，创建多节点集群对于执行本地测试的单个 DevOps 工程师来说通常是不切实际的。

幸运的是，在一台机器上安装开发 Kubernetes 集群有很多选择。这允许 DevOps 工程师在部署到共享集群之前验证他们的代码和部署过程的许多方面。

在本文中，我介绍了 DevOps 工程师在运行开发 Kubernetes 集群时可以使用的一些选项。

## 迷你库贝

minikube 是一个跨平台工具，用于创建单节点 Kubernetes 集群。它还为[提供了许多例子，展示了如何使用持续集成(CI)服务器](https://github.com/minikube-ci/examples)来配置它，允许作为自动化测试的一部分来创建和销毁短暂的测试集群。

在[在 Windows 上安装 minikube](https://octopus.com/blog/minikube-on-windows)这篇文章中学习如何在 Windows 上安装 minikube 并将其连接到 Octopus。

## 种类

[kind](https://kind.sigs.k8s.io/) ，在 Docker 中代表 Kubernetes，是一个跨平台工具，支持创建多节点 Kubernetes 集群，全部作为 Docker 容器托管。可能需要一段时间来理解 Docker 运行 Kubernetes 来编排 Docker 容器的概念，但该工具工作良好，并且比其他选项更有优势，因为它不需要 Windows 和 macOS 中的专用虚拟机。

你可以在文章[Kubernetes testing with kind](https://octopus.com/blog/kubernetes-with-kind)中找到更多关于集成 kind 集群和 Octopus 的信息。

## MicroK8s

MicroK8s 是 Windows、Linux 和 macOS 上本地开发的一个选项，也是带有企业支持和安全维护选项的生产环境的一个选项。MicroK8s 支持[插件](https://microk8s.io/docs/addons)，提供了一种方便的方式来安装常见的功能，如图像注册表、仪表盘、入口控制器等等。

## K3s

K3s 是由 Rancher 创建的一个轻量级的、生产就绪的[Kubernetes Linux](https://rancher.com/docs/k3s/latest/en/installation/installation-requirements/#operating-systems)发行版。K3s 可以用 [k3d](https://github.com/k3d-io/k3d) 在单台机器上创建多节点集群，为创建开发集群提供了便捷的解决方案。

了解如何整合牧场主和章鱼在岗位[部署到牧场主与章鱼部署](https://octopus.com/blog/deploy-to-rancher-with-octopus)。

## k0s

[k0s](https://k0sproject.io/) 是另一个轻量级的、生产就绪的[Kubernetes Linux](https://docs.k0sproject.io/v1.23.6+k0s.2/system-requirements/#host-operating-system)发行版，带有[对 Windows server](https://docs.k0sproject.io/v1.23.6+k0s.2/experimental-windows/) 的实验性支持。它内置支持[在 Docker](https://docs.k0sproject.io/v1.23.6+k0s.2/k0s-in-docker/#run-k0s-in-docker) 之上创建集群，用于 Windows、Linux 和 macOS 上的本地开发。

## Docker 桌面

[Docker 桌面](https://docs.docker.com/desktop/kubernetes/)包括一个独立的 Kubernetes 服务器和客户端，适用于 Windows、Linux 和 macOS。与这篇文章中列出的其他选项不同，Docker Desktop 是[商业软件](https://docs.docker.com/subscription/)，在某些情况下需要付费。

鉴于 Docker Desktop 是支持在 Windows 中启用 Docker 引擎的方法，许多开发人员可能已经拥有启动开发 Kubernetes 集群所需的工具，这使其成为一个方便的选择。

## 结论

由于开源社区对 Kubernetes 的热切采用，DevOps 工程师有了广泛的选择，可以在他们的本地机器上快速启动开发集群，并在 CI 测试中创建临时集群。虽然 Linux 用户有最多的选择，但对于 Windows 和 macOS 用户来说，仍然有大量活跃且维护良好的项目。因此，通常只需一两个命令和几分钟时间，DevOps 工程师就可以启动一个本地集群进行测试。

## 了解更多信息

如果你想在 AWS 平台上构建和部署容器化的应用程序，比如 EKS 和 ECS，请查看一下 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础架构代码(IaC)。

愉快的部署！