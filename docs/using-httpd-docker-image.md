# 使用 HTTPd Docker 映像- Octopus 部署

> 原文：<https://octopus.com/blog/using-httpd-docker-image>

Apache HTTP 服务器是最流行的 web 服务器之一。据维基百科报道，2022 年 3 月，它要么是使用最多的，要么是第二多的网络服务器。一张[官方 HTTPd Docker 图片](https://hub.docker.com/_/httpd)可以在 Docker Hub 上找到，已经被下载了超过 10 亿次，使其成为最受欢迎的 Docker 图片之一。

在本文中，我将向您展示如何开始使用 HTTPd 映像来托管您自己的网站或构建嵌入 HTTPd 的自定义 Docker 映像。

## 入门指南

使用 HTTPd 映像最简单的方法是让它托管来自本地工作站的静态 web 内容。将以下 HTML 代码保存到名为`index.html`的文件中:

```
<html>
    <body>
        Hello from Octopus!
    </body>
</html> 
```

然后运行 HTTPd Docker 映像，将`index.html`文件挂载到容器中的`/usr/local/apache2/htdocs/index.html`下:

```
docker run -p 8080:80 -v "$PWD/index.html":/usr/local/apache2/htdocs/index.html httpd:2.4 
```

然后您可以打开`http://localhost:8080/`查看网页。

以这种方式安装文件需要将单个 web 资产打包并分发到任何新的服务器，这很不方便。更好的解决方案是构建一个嵌入静态 web 文件的定制 Docker 映像。

## 基于 HTTPd 创建自定义图像

要创建自己的 Docker 图像，请将以下文本保存到名为`Dockerfile`的文件中:

```
FROM httpd:2.4
COPY index.html /usr/local/apache2/htdocs/index.html 
```

`Dockerfile`包含构建自定义 Docker 映像的说明。这里您使用`FROM`命令将您的映像基于 HTTPd 映像，然后使用`COPY`命令将您的`index.html`文件复制到`/usr/local/apache2/htdocs`目录下的新映像中。

使用以下命令构建新映像:

```
docker build . -t myhttpd 
```

这构建了一个名为`myhttpd`的新图像。使用以下命令运行新映像:

```
docker run -p 8080:80 myhttpd 
```

注意，这次您没有挂载任何目录。然而，当你打开`http://localhost:8080/index.html`时，你的定制 HTML 页面会显示出来，因为它嵌入在你的定制图像中。

HTTPd 不仅仅能够托管静态网站。要释放 HTTPd 的全部潜力，您必须添加定制的配置文件。

## 高级 HTTPd 配置

使用以下命令提取 HTTPd 映像中嵌入的标准配置文件:

```
docker run --rm httpd:2.4 cat /usr/local/apache2/conf/httpd.conf > my-httpd.conf 
```

生成的文件很大，有 500 多行代码，所以我不会在这里列出。

要让 HTTPd 加载额外的配置文件，可以在该文件的末尾添加一条指令，指示 HTTPd 加载指定目录中的文件:

```
IncludeOptional conf/sites/*.conf 
```

用以下内容创建一个名为`health-check.conf`的文件。这个配置使 HTTPd 能够监听端口 90，并用 200 OK 响应来响应`/health`路径上的请求。此响应模拟 web 服务器的运行状况检查，客户端可以使用该检查来确定服务器是否启动并运行:

```
LoadModule rewrite_module modules/mod_rewrite.so
Listen 90

<VirtualHost *:90>
  <Location /health>
      ErrorDocument 200 "Healthy"
      RewriteEngine On
      RewriteRule .* - [R=200]
  </Location>
</VirtualHost> 
```

然后更新`DockerFile`以创建目录`/usr/local/apache2/conf/sites/`，将`health-check.conf`文件复制到该目录，并用包含`IncludeOptional`指令的副本覆盖原始配置文件:

```
FROM httpd:2.4
RUN mkdir -p /usr/local/apache2/conf/sites/
COPY health-check.conf /usr/local/apache2/conf/sites/health-check.conf
COPY my-httpd.conf /usr/local/apache2/conf/httpd.conf 
```

使用以下命令构建新映像:

```
docker build . -t myhttpd 
```

使用命令运行 Docker 映像。请注意，您公开了一个新端口来提供对运行状况检查端点的访问:

```
docker run -p 8080:80 -p 9090:90 myhttpd 
```

然后打开[http://localhost:9090/health](http://localhost:9090/health)进入健康检查页面。

## 选择 HTTPd 变体

HTTPd 图像基于 Debian 或 Alpine。Alpine 经常被用作 Docker 图像的轻量级基础。要查看 Docker 图像的大小，必须首先将它们下载到您的本地工作站:

```
docker pull httpd:2.4
docker pull httpd:2.4-alpine 
```

然后，您可以使用以下命令查找图像大小:

```
docker image ls 
```

从这里你可以看到 Alpine 的变体在尺寸上要小得多，基于 Debian 的 imaged 重 145MB，Alpine image 重 55MB。

要使用 Alpine 变体，请将您的自定义图像基于`httpd:2.4-alpine`图像:

```
FROM httpd:2.4-alpine
RUN mkdir -p /usr/local/apache2/conf/sites/
COPY health-check.conf /usr/local/apache2/conf/sites/health-check.conf
COPY my-httpd.conf /usr/local/apache2/conf/httpd.conf 
```

## 结论

HTTPd 是一个流行的 web 服务器，官方的 HTTPd Docker 映像允许 DevOps 团队在 Docker 中托管定制的 web 应用程序。通过对 HTTPd 配置文件进行一些小的调整，还可以使用全系列的 [HTTPd 模块](https://httpd.apache.org/docs/current/mod/)，解锁许多高级功能。

在本文中，您了解了如何创建托管静态 web 应用程序的自定义 Docker 映像，添加了高级 HTTPd 配置文件来提供健康检查端点，并比较了 Debian 和 Alpine HTTPd 映像的大小。

您可能还想了解:

## 资源

## 了解更多信息

如果您想在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，请尝试使用 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础架构代码(IaC)。

愉快的部署！