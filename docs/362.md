# 章鱼部署 Java 支持 RFC -章鱼部署

> 原文：<https://octopus.com/blog/java-rfc>

章鱼最初是用。NET 开发人员的想法，并附带了许多约定，使部署。网络应用简单。在基本层面上，Octopus 提供了作为传输层的触手，以及执行 PowerShell 的能力。在此基础之上，还有将应用程序部署到 IIS 或设置 Windows 服务的内置步骤，以及需要处理的约定。NET 配置文件或转换。

当然，你已经可以使用八达通不仅仅是。NET 应用程序——您可以压缩一个 Java 应用程序，并将其推送到 Windows 机器上，然后使用 PowerShell 来配置它。或者最近你可以用 SSH 和 Bash 做同样的事情。这足以让 Java 部署工作——我最近展示了如何部署到 [WildFly](https://octopus.com/blog/wildfly-deploy) 和 [Tomcat](https://octopus.com/blog/octopus-tomcat) ，以及将 [Spring Boot 应用程序部署为 Windows 服务](https://octopus.com/blog/spring-boot-windows-services)。

你可以让它工作，但是目前还没有任何 Java 应用程序的高级约定。你可以说 Octopus 目前支持 Java 应用程序，但不像它支持的那样，为它们提供一流的支持。网络应用。

## Octopus 中一流的 Java 支持

今天，我们宣布我们的长期愿景，让我们的 Java 部署故事与我们的。NET 部署故事——我们希望它们都是平等的、一流的体验。今天的 RFC 着眼于我们最初的计划，看看这意味着什么——我们计划很快构建什么——但我们也希望从您那里得到指导:Octopus 中一流的 Java 支持对您来说意味着什么？

## 目标

本 RFC 中概述的特性的目标是为在 Windows 和 Linux 上运行的 Tomcat 和 wildly/JBoss EAP 部署 Java 应用程序提供一流的支持。

## 应用服务器支持

最初 Octopus Deploy 将支持 Tomcat 和 WildFly / JBoss EAP。选择这些应用服务器是因为它们有很大的市场份额，当结合起来时，它们代表了生产中大约三分之二的 Java 服务器。

我们将针对红帽 JBoss EAP 6 及以上，Wildfly 10 及以上，Tomcat 7 及以上。

我们不可能一开始就实现对所有应用服务器的支持，但是如果你希望看到对 WebSphere、WebLogic、Glassfish、Payara 等的支持。请添加评论。

支持这些应用服务器意味着向 Octopus Deploy 添加以下步骤:

*   部署包
*   文件复制(即将 WAR 文件复制到`webapps`或`deployments`目录，或者分解 WAR 文件)
*   通过 Tomcat 管理器进行 HTTP 上传
*   使用 WildFly CLI 工具进行 CLI 部署
*   证书管理
*   导出 JKS Java 密钥库
*   在 WildFly 或 Tomcat 中配置密钥库
*   创建和部署 WildFly 保管库以存储可在 WildFly 配置文件中引用的敏感信息。

## 配置支持

为了支持 Java 应用程序中的各种配置文件，将添加一些步骤，以便能够转换 Java 包中任意位置的 XML、YAML 和属性文件。这将允许 Octopus Deploy 支持 Spring 和 Java Servlet 或 JavaEE 应用程序广泛使用的 XML 配置文件，以及 [Spring 外部化配置](https://docs.spring.io/spring-boot/docs/current/reference/html/boot-features-external-config.html)。

## Linux 支持

由于 Linux 很受 Java 开发人员的欢迎，部署服务的现有步骤将被扩展到同时支持 init 和 systemd。

目前我们要求 Mono 安装在 Linux 服务器上。我们已经把乌贼和触手的部分转移到。NET 核心，所以这应该消除 Mono 依赖。

## 包装支持

目前 Octopus Deploy 接受 ZIP 和 Nuget 包。为了支持 Java，将本机支持 JAR、WAR、EAR 和 RAR 包格式。这将允许上传 Java 包，而不需要先将它们打包成 ZIP 文件。

Octopus 还将支持 Maven 存储库作为一级包提要，这是目前支持 NuGet 和 Docker Registry 的方式。

## 版本支持

将添加对 Maven 版本控制方案的支持，以允许 Octopus Deploy 比较两个 Java 包的版本。版本信息将在`Implementation-Version`键下的`META-INF/manifest.mf`文件中找到。

## 反馈

如果您对我们提议的更改有任何建议或意见，请留下您的评论。如果您希望看到 Octopus Deploy 支持 Java 部署场景，或者如果您有特定的功能需求，您的反馈是很有价值的。