# 使用 Alpine Docker 图像- Octopus 部署

> 原文：<https://octopus.com/blog/using-alpine-docker-image>

由于尺寸较小， [Alpine Docker 图像](https://hub.docker.com/_/alpine)经常被用作其他自定义图像的基础。在 Docker Hub 上有超过 10 亿的下载量，Alpine 也是最受欢迎的图片之一。(你也可以阅读我在[发的关于 Ubuntu](https://octopus.com/blog/using-ubuntu-docker-image) 的帖子，它目前是 Docker Hub 下载最多的图片。)

在这篇文章中，我向你展示了当你在 Alpine 上创建自己的图片时可以采用的最佳实践。

## 清理缓存

在工作站和服务器上安装软件时，缓存的软件包列表非常有用，因为它们提供了对软件包存储库中可用软件包的快速访问。然而，安装在 Docker 容器中的包很少在运行时更新；相反，Docker 映像本身被更新，容器被重新创建。这意味着将包缓存列表烘焙到 Docker 映像中是不必要且低效的。

删除缓存包列表的一种方法是用`apk add --update-cache`命令安装新的包，然后删除`/var/cache/apk`下的文件，作为单个`RUN`指令的一部分。这可确保创建软件包缓存，这是安装其他软件包所必需的，然后在不捕获中间映像层中的软件包缓存的情况下进行清理:

```
RUN apk add --update-cache \
    python \
    python-dev \
    py-pip \
    build-base \
  && pip install virtualenv \
  && rm -rf /var/cache/apk/* 
```

您也可以使用`apk add --no-cache`选项。这相当于前面的命令，但更简洁:

```
RUN apk add --no-cache nginx 
```

## 虚拟包

虚拟包提供了一种将包捆绑在一个通用名称下的方法，允许它们作为一个组被删除。 [Alpine Docker 图像文档](https://github.com/alpinelinux/docker-alpine/blob/master/docs/usage.adoc)提供了安装 Python 开发库、下载 Python 应用程序的依赖项，然后移除 Python 开发库的示例:

```
FROM alpine

WORKDIR /myapp
COPY . /myapp

RUN apk add --no-cache python py-pip openssl ca-certificates
RUN apk add --no-cache --virtual build-dependencies python-dev build-base wget \
  && pip install -r requirements.txt \
  && python setup.py install \
  && apk del build-dependencies

CMD ["myapp", "start"] 
```

虽然这个例子可行，但是效率很低。由于 Docker 缓存的实现方式，对使用指令`COPY . /myapp`复制到映像中的文件的任何更改都会使缓存失效，迫使后续指令重新运行。实际上，这意味着上面的例子将在每次 Python 代码改变时下载、安装和删除 Python 开发库。

更好的解决方案是使用[多阶段构建](https://docs.docker.com/develop/develop-images/multistage-build/)。下面显示了一个示例:

```
FROM alpine AS compile-image

RUN apk add --no-cache python3 py-pip openssl ca-certificates python3-dev build-base wget

WORKDIR /myapp

COPY requirements.txt /myapp/
RUN python3 -m venv /myapp
RUN /myapp/bin/pip install -r requirements.txt

FROM alpine AS runtime-image

RUN apk add --no-cache python3 openssl ca-certificates

WORKDIR /myapp
COPY . /myapp

COPY --from=compile-image /myapp/ ./

CMD ["/myapp/bin/python", "myapp.py", "start"] 
```

多阶段构建允许您使用构建应用程序源代码所需的开发库来创建映像。这个“编译映像”在两次构建之间保留了这些开发库，消除了每次下载它们的需要。

创建第二个映像来托管可执行的应用程序代码和运行时库，但具体来说不包括任何仅在编译时需要的库。这个“运行时映像”尽可能小，因为它复制由编译映像产生的文件，而不需要相关的编译时库。

## musl vs glibc

在很大程度上，Alpine 可以作为任何其他基本 Docker 图像的替代物。然而，了解 Alpine 和其他常见的基础 Docker 映像(如 Ubuntu、Debian 或 Fedora)之间的架构差异非常重要。

Alpine 使用 [musl C 标准库](http://musl.libc.org/)，而 Ubuntu、Debian、Fedora 使用 [glibc](https://www.gnu.org/software/libc/) 。下面是[两个库](http://www.etalabs.net/compare_libcs.html)的详细对比。

在某些情况下，第三方工具假设或需要 glibc。例如， [Visual Studio 代码远程容器执行文档](https://code.visualstudio.com/docs/remote/containers)提供了以下警告:

> 当使用 Alpine Linux 容器时，由于扩展中本机代码的 glibc 依赖性，一些扩展可能无法工作。

基于 Alpine 的图像也[不适合用作 Octopus 容器图像](https://octopus.com/docs/projects/steps/execution-containers-for-workers):

> 构建在 musl 上的 Linux 发行版，尤其是 Alpine，不支持 Calamari，也不能用作容器映像。这是因为 Calamari 目前只针对 glibc 编译，而不是 musl。

PythonSpeed 的博客文章[使用 Alpine 可以让 Python Docker 构建速度慢 50 倍](https://pythonspeed.com/articles/alpine-docker-python/)，详细介绍了在 Alpine 上构建 Python 应用时的一些性能问题:

> 大多数 Linux 发行版使用标准 C 库的 GNU 版本(glibc ),几乎每个 C 程序都需要它，包括 Python。但是 Alpine Linux 使用的是 musl，那些二进制轮子是针对 glibc 编译的，因此 Alpine 禁用了 Linux 轮子支持。

除了已知与 musl 不兼容的特定用例之外，我发现 Alpine 是一个可靠而实用的选择，可以作为我自己的 Docker 图像的基础。但是了解使用实现 musl 的发行版的含义是有好处的。

## 结论

Alpine 提供了一个轻量级和流行的 Docker 映像，与其他流行的映像(如 Ubuntu)相比，它可以改进您的映像构建和部署时间。

Alpine 使用 musl C 标准库，这在某些情况下可能会引入兼容性问题，但是您通常可以假设 Alpine 为您的定制 Docker 映像提供了所需的一切。虚拟包等高级功能也可以让您缩小映像大小，尽管多阶段构建可能是更好的选择。

了解如何使用其他流行的容器图像:

## 资源

## 了解更多信息

如果您想在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，请尝试使用 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础架构代码(IaC)。

愉快的部署！