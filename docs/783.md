# 使用 Ubuntu Docker 镜像- Octopus 部署

> 原文：<https://octopus.com/blog/using-ubuntu-docker-image>

Ubuntu Docker 官方图片是 Docker Hub 下载最多的图片。超过 10 亿的下载量证明了 Ubuntu 是一个受欢迎和可靠的基础映像，你可以在它的基础上构建自己的定制 Docker 映像。

在这篇文章中，我将向你展示如何在构建你自己的 Docker 映像时充分利用基本的 Ubuntu 映像。

## Dockerfile 文件示例

这是一个例子`Dockerfile`,包括在这篇文章中讨论的调整。我仔细检查了每个设置，以解释它们增加了什么价值:

```
FROM ubuntu:22.04
RUN echo 'APT::Install-Suggests "0";' >> /etc/apt/apt.conf.d/00-docker
RUN echo 'APT::Install-Recommends "0";' >> /etc/apt/apt.conf.d/00-docker
RUN DEBIAN_FRONTEND=noninteractive \
  apt-get update \
  && apt-get install -y python3 \
  && rm -rf /var/lib/apt/lists/*
RUN useradd -ms /bin/bash apprunner
USER apprunner 
```

使用以下命令构建映像:

```
docker build . -t myubuntu 
```

现在你已经看到了如何从 Ubuntu 基础映像构建一个自定义映像，让我们仔细检查每一个设置来理解为什么要添加它们。

## 选择基础图像

Docker 映像适用于所有版本的 Ubuntu，包括长期支持(LTS)版本，如 20.04 和 22.04，以及普通版本，如 19.04、19.10、21.04 和 21.10。

LTS 版本支持 5 年，在此期间相关的 Docker 映像也由 Canonical 维护，如 [Ubuntu 发布周期页面](https://ubuntu.com/about/release-cycle)所述:

> 这些映像也会随着安全更新映像的定期发布而保持更新，您应该自动使用最新的映像，以确保为您的用户提供一致的安全保护。

当创建托管生产软件的 Docker 映像时，以最新的 LTS 版本为基础是有意义的。这允许开发运维团队在最新的 LTS 基础映像的基础上重建他们的定制映像，该映像自动包括所有更新，但也不太可能包括主要操作系统版本之间可能引入的那种重大更改。

我使用了 Ubuntu 22.04 LTS Docker 图像作为这个图像的基础:

```
FROM ubuntu:22.04 
```

## 没有安装建议或推荐的依赖项

一些软件包有一个建议或推荐的依赖项列表，这些依赖项不是必需的，但在默认情况下是安装的。这些额外的依赖会不必要地增加最终 Docker 图像的大小，正如 Ubuntu 在他们关于减小 Docker 图像大小的[博客文章中提到的](https://ubuntu.com/blog/we-reduced-our-docker-images-by-60-with-no-install-recommends)。

为了禁用对所有`apt-get`调用的这些可选依赖项的安装，在`/etc/apt/apt.conf.d/00-docker`处用以下设置创建配置文件:

```
RUN echo 'APT::Install-Suggests "0";' >> /etc/apt/apt.conf.d/00-docker
RUN echo 'APT::Install-Recommends "0";' >> /etc/apt/apt.conf.d/00-docker 
```

## 安装附加软件包

大多数基于 Ubuntu 的定制镜像需要你安装额外的包。例如，要运行用 Python、PHP、Java、Node.js 或 DotNET 编写的自定义应用程序，您的自定义映像必须安装与这些语言相关的包。

在典型的工作站或服务器上，使用简单的命令安装软件包，如:

```
apt-get install python3 
```

在 Docker 映像中安装新软件的过程是非交互式的，这意味着您没有机会响应提示。这意味着您必须添加`-y`参数，以便在提示继续安装软件包时自动回答“是”:

```
RUN apt-get install -y python3 
```

## 防止软件包安装过程中出现提示错误

一些软件包的安装试图打开额外的提示，以进一步自定义安装选项。在非交互环境中，例如在 Docker 映像的构建过程中，试图打开这些对话框会导致如下错误:

```
unable to initialize frontend: Dialog 
```

这些错误可以忽略，因为它们不会阻止软件包的安装。但是可以通过将`DEBIAN_FRONTEND`环境变量设置为`noninteractive`来防止这些错误:

```
RUN DEBIAN_FRONTEND=noninteractive apt-get install -y python3 
```

Docker 网站提供了关于 DEBIAN _ FRONTEND 环境变量使用的官方指导。他们认为这是一种装饰性的改变，并建议不要永久设置环境变量。上面的命令为单个`apt-get`命令的持续时间设置了环境变量，这意味着对`apt-get`的任何后续调用都不会定义`DEBIAN_FRONTEND`。

## 清理包列表

在安装任何软件包之前，您需要通过调用以下命令来更新软件包列表:

```
RUN apt-get update 
```

但是，在安装了所需的软件包之后，软件包列表就没有什么价值了。最佳实践是从 Docker 映像中删除任何不必要的文件，以确保生成的映像尽可能小。安装完所需的软件包后，为了清理软件包列表，删除了`/var/lib/apt/lists/`下的文件。

在这里，您可以更新软件包列表，安装所需的软件包，并作为一个命令的一部分清理软件包列表，该命令分为多行，每行末尾有一个反斜杠:

```
RUN DEBIAN_FRONTEND=noninteractive \
  apt-get update \
  && apt-get install -y python3 \
  && rm -rf /var/lib/apt/lists/* 
```

## 以非超级用户身份运行

默认情况下，root 用户在 Docker 容器中运行。root 用户通常拥有比运行自定义应用程序所需更多的权限，因此创建没有 root 权限的新用户可以提供更好的安全性。

`useradd`命令[提供了创建新用户](https://manpages.ubuntu.com/manpages/jammy/en/man8/useradd.8.html)的非交互方式。

这不要与`adduser`命令混淆，后者是比`useradd`更高级别的[包装器](https://manpages.ubuntu.com/manpages/jammy/en/man8/adduser.8.html)。

在编辑了所有配置文件并安装了软件包之后，您创建了一个名为`apprunner`的新用户:

```
RUN useradd -ms /bin/bash apprunner 
```

然后，该用户将被设置为任何进一步操作的默认用户:

```
USER apprunner 
```

## 结论

除了安装任何需要的额外的软件包之外，很少定制就可以使用基本的 Ubuntu Docker 镜像。但是，通过一些调整来限制安装可选软件包，在软件包安装后清理软件包列表，并创建具有有限权限的新用户来运行自定义应用程序，您可以为您的自定义应用程序创建更小、更安全的映像。

了解如何使用其他流行的容器图像:

## 资源

## 了解更多信息

如果您想在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，请尝试使用 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础架构代码(IaC)。

愉快的部署！