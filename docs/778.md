# 使用 NGINX Docker 映像- Octopus 部署

> 原文：<https://octopus.com/blog/using-nginx-docker-image>

Docker 是一个打包和运行 web 应用程序的引人注目的平台，尤其是当它与云平台提供的众多平台即服务(PaaS)产品之一结合使用时。NGINX 长期以来为 DevOps 团队提供了在 Linux 上托管 web 应用程序的能力，并且还提供了一个官方的 Docker 映像作为定制 web 应用程序的基础。

在这篇文章中，我解释了 DevOps 团队如何使用 [NGINX Docker 映像](https://hub.docker.com/_/nginx)在 Docker 上构建和运行 web 应用程序。

## 基础映像入门

NGINX 是一个多用途的工具，包括负载平衡器、反向代理和网络缓存。然而，当在 Docker 容器中运行 NGINX 时，这些高级功能中的大部分被委托给其他专门的平台或 NGINX 的其他实例。通常，当运行在 Docker 容器中时，NGINX 实现了 web 服务器的功能。

要使用默认网站创建 NGINX 容器，请运行以下命令:

```
docker run -p 8080:80 nginx 
```

该命令将下载`nginx`映像(如果尚未下载)并创建一个容器，将容器中的端口 80 暴露给主机上的端口 8080。然后可以打开`http://localhost:8080/index.html`查看默认的“欢迎使用 nginx！”网站。

为了允许 NGINX 容器公开定制的 web 资产，可以在 Docker 容器中挂载一个本地目录。

将以下 HTML 代码保存到名为`index.html`的文件中:

```
<html>
    <body>
        Hello from Octopus!
    </body>
</html> 
```

接下来，运行下面的命令，在 NGINX 容器中的`/usr/share/nginx/html`下挂载当前目录，并进行只读访问:

```
docker run -v $(pwd):/usr/share/nginx/html:ro -p 8080:80 nginx 
```

再次打开`http://localhost:8080/index.html`,您会看到显示的自定义 HTML 页面。

Docker 映像的一个好处是能够将所有相关文件捆绑到一个可分发的工件中。要实现这一优势，您必须基于 NGINX 映像创建一个新的 Docker 映像。

## 基于 NGINX 创建自定义图像

要创建自己的 Docker 图像，请将以下文本保存到名为`Dockerfile`的文件中:

```
FROM nginx
COPY index.html /usr/share/nginx/html/index.html 
```

`Dockerfile`包含构建自定义 Docker 映像的说明。在这里，您使用`FROM`命令将您的映像基于 NGINX one，然后使用`COPY`命令将您的`index.html`文件复制到`/usr/share/nginx/html`目录下的新映像中。

使用以下命令构建新映像:

```
docker build . -t mynginx 
```

这构建了一个名为`mynginx`的新图像。使用以下命令运行新映像:

```
docker run -p 8080:80 mynginx 
```

注意，这次您没有挂载任何目录。然而，当你打开`http://localhost:8080/index.html`时，你的定制 HTML 页面会显示出来，因为它嵌入在你的定制图像中。

NGINX 不仅仅能够托管静态文件。要解锁此功能，您必须使用自定义 NGINX 配置文件。

## 高级 NGINX 配置

NGINX 通过配置文件公开其功能。默认的 NGINX 映像附带了一个简单的默认配置文件，用于托管静态 web 内容。该文件位于默认图像中的`/etc/nginx/nginx.conf`处，其内容如下:

```
user  nginx;
worker_processes  auto;

error_log  /var/log/nginx/error.log notice;
pid        /var/run/nginx.pid;

events {
    worker_connections  1024;
}

http {
    include       /etc/nginx/mime.types;
    default_type  application/octet-stream;

    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile        on;
    #tcp_nopush     on;

    keepalive_timeout  65;

    #gzip  on;

    include /etc/nginx/conf.d/*.conf;
} 
```

不需要详细理解这个配置文件，但是有一行有趣的内容指示 NGINX 从`/etc/nginx/conf.d`目录加载额外的配置文件:

```
include /etc/nginx/conf.d/*.conf; 
```

默认的`/etc/nginx/conf.d`文件将 NGINX 配置为 web 服务器。具体来说，`location /`阻止从`/usr/share/nginx/html`加载文件是你之前将 HTML 文件挂载到那个目录的原因:

```
server {
    listen       80;
    server_name  localhost;

    #access_log  /var/log/nginx/host.access.log  main;

    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
    }

    #error_page  404              /404.html;

    # redirect server error pages to the static page /50x.html
    #
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }

    # proxy the PHP scripts to Apache listening on 127.0.0.1:80
    #
    #location ~ \.php$ {
    #    proxy_pass   http://127.0.0.1;
    #}

    # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
    #
    #location ~ \.php$ {
    #    root           html;
    #    fastcgi_pass   127.0.0.1:9000;
    #    fastcgi_index  index.php;
    #    fastcgi_param  SCRIPT_FILENAME  /scripts$fastcgi_script_name;
    #    include        fastcgi_params;
    #}

    # deny access to .htaccess files, if Apache's document root
    # concurs with nginx's one
    #
    #location ~ /\.ht {
    #    deny  all;
    #}
} 
```

您可以利用该指令来加载`/etc/nginx`中的任何`*.conf`配置文件，以定制 NGINX。在本例中，您添加了一个健康检查，它通过一个自定义位置监听端口 90，用 HTTP 200 OK 响应对`/nginx-health`路径的请求。

将以下文本保存到名为`health-check.conf`的文件中:

```
server {
    listen       90;
    server_name  localhost;

    location /nginx-health {
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
} 
```

修改`Dockerfile`将配置文件复制到`/etc/nginx/conf.d`:

```
FROM nginx
COPY index.html /usr/share/nginx/html/index.html
COPY health-check.conf /etc/nginx/conf.d/health-check.conf 
```

使用以下命令构建映像:

```
docker build . -t mynginx 
```

使用命令运行新映像。请注意 9090 上显示的新端口:

```
docker run -p 8080:80 -p 9090:90 mynginx 
```

现在打开`http://localhost:9090/nginx-health`。返回运行状况检查响应，表明 web 服务器已启动并正在运行。

以上示例基于默认的`nginx`图像定制您的图像。但是也有其他的变体，在不牺牲任何功能的情况下提供更小的图像尺寸。

## 选择 NGINX 变体

默认的`nginx`图像基于 [Debian](https://github.com/nginxinc/docker-nginx/blob/master/Dockerfile-debian.template) 。不过 NGINX 也提供基于[阿尔卑斯](https://github.com/nginxinc/docker-nginx/blob/master/Dockerfile-alpine.template)的图片。

Alpine 经常被用作 Docker 图像的轻量级基础。要查看 Docker 图像的大小，必须首先将它们下载到您的本地工作站:

```
docker pull nginx
docker pull nginx:stable-alpine 
```

然后，您可以使用以下命令查找图像大小:

```
docker image ls 
```

从这里你可以看到 Debian 图像重约 140 MB，而 Alpine 图像重约 24 MB。这大大节省了图像尺寸。

为了让您的图像基于 Alpine 变体，您需要更新`Dockerfile`:

```
FROM nginx:stable-alpine
COPY index.html /usr/share/nginx/html/index.html
COPY health-check.conf /etc/nginx/conf.d/health-check.conf 
```

使用以下命令构建并运行映像:

```
docker build . -t mynginx
docker run -p 8080:80 -p 9090:90 mynginx 
```

再次打开`http://localhost:9090/nginx-health`或`http://localhost:8080/index.html`查看网页。一切都像以前一样继续工作，但是您的自定义映像现在要小得多。

## 结论

NGINX 是一个强大的 web 服务器，官方的 NGINX Docker 映像允许 DevOps 团队在 Docker 中托管定制的 web 应用程序。NGINX 还支持高级场景，因为它能够读取复制到自定义 Docker 映像中的配置文件。

在本文中，您了解了如何创建托管静态 web 应用程序的自定义 Docker 映像，添加了高级 NGINX 配置文件来提供健康检查端点，并比较了 Debian 和 Alpine NGINX 映像的大小。

了解如何使用其他流行的容器图像:

## 资源

## 了解更多信息

如果您想在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，请尝试使用 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础架构代码(IaC)。

愉快的部署！