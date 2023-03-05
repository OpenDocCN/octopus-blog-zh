# 自动触手安装-章鱼部署

> 原文：<https://octopus.com/blog/automated-tentacle-installation>

触手安装器由两部分组成:

1.  MSI，它将文件安装到 C:\Program Files
2.  一个配置工具，它安装 Windows 服务，设置信任关系，让您配置端口号，等等

配置工具是独立的，因为它可以随时重新运行，这类似于通过 MSI 安装 SQL Server Reporting Services 的方式，但随后通过 Reporting Services 配置管理器进行配置。

MSI 可以通过组策略或其他企业软件安装工具轻松部署，但配置工具是一个 GUI，必须手动运行。

在 Octopus [聊天室](http://jabbr.net/#/rooms/octopus "Octopus chat")， [justFen_](https://twitter.com/justfen_ "justFen_") 建议应该可以在完全无人值守的情况下安装触手。从版本 **1.0.19.1298** 开始，现在这是可能的。配置触手由几个命令组成，我将一一介绍。

**更新:此功能的文档已被[移至文档页面](http://octopusdeploy.com/documentation/configuration/installation)。**

昨晚，我与一位拥有超过 100 个触须的客户进行了交谈，每个触须都是手工精心安装的。虽然我怀疑人们在安装少量的触手服务器时仍然会使用 GUI 安装程序，但是我认为这个特性在管理大量的部署代理时会非常方便。