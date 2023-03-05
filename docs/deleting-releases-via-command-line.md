# 通过命令行删除发布- Octopus Deploy

> 原文：<https://octopus.com/blog/deleting-releases-via-command-line>

我们的待办事项中有一项是能够[为项目设置一个保留策略](https://trello.com/card/auto-purge-tentacles/4e907de70880ba000079b75c/20)，它可以自动清理旧的部署应用程序、缓存的 NuGet 包，以及来自 Octopus UI 的发布/部署。

这一项还没有完成，但与此同时，如果你需要删除发布/部署，你现在可以使用最新版本的 Octo.exe 和运行 1.0.31 或更高版本的 T2 章鱼服务器来完成。

在命令行中，语法是:

```
octo delete-releases --project=MyProject --minversion=1.0.0 --maxversion=2.0.0 --apikey=ABCDEF... --server=http://<your-octopus> 
```

版本号包含在内，可以部分指定，例如`2.0.0`或`2.0.1982.12981`。[比较版本号时使用 SemVer](http://semver.org/) 规则。

正如我所说的，这还不足以成为一个完整的保留策略特性，但是如果您由于构建/部署脚本中的循环依赖而意外地创建了几百个版本，这可能是有用的:)