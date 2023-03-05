# 采访:部署 NuGet.org-章鱼部署

> 原文：<https://octopus.com/blog/deploying-nuget.org>

支持 NuGet 包管理器的网络画廊和基础设施 NuGet 已经成为。网络开发。很难想象没有 NuGet 的时代，那时使用开源库意味着搜索网页、下载 ZIP 文件和手动添加参考资料。NuGet 也是 Octopus Deploy 所依赖的基础，因为我们使用它作为我们的应用程序打包格式。

使用 NuGet 支持的部署工具来部署 NuGet 背后的服务是一件非常酷的事情！来自微软的安德鲁·斯坦顿护士非常友好地带我参观了 NuGet 团队如何使用 Octopus Deploy，这是我们在这个 22 分钟的视频中记录的。

NuGet.org 运行在 Windows Azure 之上。他们有三种不同的云服务，并且需要开发、测试和生产环境，总共需要 9 个云服务终端。当你有一两个服务时，从 Visual Studio 发布 Azure 包是可行的，但一旦你管理 9 个服务，就没什么意思了。观看视频，了解他们如何使用 Octopus Deploy 来自动化其 Azure 部署！