# 如何用 Octopus Deploy 安装和配置 NGINX

> 原文：<https://octopus.com/blog/install-and-configure-nginx>

你好，部署者！在过去的几天里，我学到了很多关于配置 NGINX 的知识。我已经为两个 web 应用程序配置了一个简单的反向代理，希望以后再也不要这样做了。为了我未来的自己，我在 Octopus 中建立了一个项目来做这项工作。配置更改、证书到期或部署到另一台机器都可以通过一个按钮来完成，而不是试图记住我在过去几天里所做的所有调整。它是这样工作的:您好部署者！在过去的几天里，我学到了很多关于配置 NGINX 的知识。我已经为两个 web 应用程序配置了一个简单的反向代理，希望以后再也不要这样做了。为了我未来的自己，我在 Octopus 中建立了一个项目来做这项工作。配置更改、证书到期或部署到另一台机器都可以通过一个按钮来完成，而不是试图记住我在过去几天里所做的所有调整。它是这样工作的:

[![NGINX deployment process](img/68328a5c6d664d4596388d6a0d0b9725.png)](#)

## 安装 NGINX

要使用 NGINX，我们必须安装 NGINX。一个简单的 bash 脚本确保安装了 NGINX:

```
sudo apt-get install --assume-yes nginx 
```

## 安装站点配置

有两个配置文件指导 NGINX 如何将请求转发到我们的 web 应用程序。它们看起来像这样:

```
server {
    listen 80; listen [::]:80;
    server_name somewhere.octopus.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;  listen [::]:443 ssl http2;
    server_name somewhere.octopus.com;

    ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
    ssl on;
    ssl_certificate      /etc/ssl/octopus.crt;
    ssl_certificate_key  /etc/ssl/octopus.key;
    ssl_session_tickets off;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-RSA-AES128-SHA256:ECDHE-RSA-AES128-SHA:ECDHE-RSA-AES256-SHA384:ECDHE-RSA-AES256-SHA;
    ssl_prefer_server_ciphers on;

    http2_idle_timeout 5m; # up from 3m default

    location / {
        proxy_pass http://unix:/var/discourse/shared/standalone/nginx.http.sock:;
        proxy_set_header Host $http_host;
        proxy_http_version 1.1;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto https;
    }
} 
```

这些文件存在于源代码控制中，并被打包供 Octopus 使用。这一步将文件复制到它们在`sites-available` NGINX 配置目录中的目的地，然后在`sites-enabled`目录中创建一个符号链接。当 NGINX 启动时，它从`sites-enabled`目录加载配置，并开始向这些站点转发请求。有一个部署后 bash 脚本，类似于:

```
sudo rm /etc/nginx/sites-enabled/*
sudo rm /etc/nginx/sites-available/*

sudo cp somewhere.nginx /etc/nginx/sites-available/somewhere
sudo ln -s /etc/nginx/sites-available/somewhere /etc/nginx/sites-enabled/somewhere 
```

## 安装 SSL 证书

Octopus 对 SSL 证书有一流的支持。该步骤将 SSL 证书和密钥文件复制到上面站点配置中指定的位置(/etc/ssl)。NGINX 需要一个用于密钥的 PEM 文件和一个用于证书的 PEM 文件。证书文件必须包含整个证书链，即主证书和任何中间证书。为了让 Octopus 以正确的格式提供证书，我用主证书、中间证书和私钥创建了一个 PEM 文件。它看起来像这样:

```
-----BEGIN CERTIFICATE-----
PRIMARY_CERTIFICATE
-----END CERTIFICATE-----
-----BEGIN CERTIFICATE-----
INTERMEDIATE_CERTIFICATE
-----END CERTIFICATE-----
-----BEGIN RSA PRIVATE KEY-----
PRIVATE_KEY
-----END RSA PRIVATE KEY----- 
```

证书在八达通证书图书馆。该步骤有一个名为`sslCert`的变量，它引用库证书。在该步骤中，变量用于创建证书和密钥文件，然后将它们复制到/etc/ssl 目录中各自的位置。注意`RawOriginal`用于维护证书链:

```
KEY=$(get_octopusvariable "sslCert.PrivateKeyPem")
echo "$KEY" > ssl.key
sudo mv ssl.key /etc/ssl/octopus.key

CERT=$(get_octopusvariable "sslCert.RawOriginal")
echo "$CERT" | base64 -d > ssl.crt
sudo mv ssl.crt  /etc/ssl/octopus.crt 
```

## 正在启动 NGINX

最后，是时候启动(或重启)NGINX 了。这一步是重启 NGINX 服务的简单 bash 脚本:

```
sudo service nginx restart 
```

## 结论

随着可重复部署过程的建立，我现在可以对配置文件进行更改或更新证书，而不必花费数小时重新学习如何配置 NGINX。我的一个同事可以拿起这个项目，看看 NGINX 是如何配置的，这样他们就可以在几分钟内开始做贡献，而不是费力地阅读文档或试图对我所做的进行逆向工程。最棒的是，如果老板让我重新配置 NGINX，我可以重用我的 Octopus Deploy 项目，然后在海滩上度过一天的剩余时间。愉快的部署！