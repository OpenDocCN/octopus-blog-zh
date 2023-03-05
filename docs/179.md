# 部署失败。ps1 支持- Octopus 部署

> 原文：<https://octopus.com/blog/deployfailed>

今天的 Octopus Deploy 版本包括对一个`DeployFailed.ps1`脚本的支持，这个特性我在最近的一篇关于 Octopus 如何处理回滚的博文[中提到过。](http://octopusdeploy.com/blog/rollback)

### 背景

在部署过程中，触手通常会运行以下步骤:

1.  从 NuGet 中提取包
2.  运行`PreDeploy.ps1`
3.  运行 XML 配置转换
4.  替换 XML 配置`appSettings`和`connectionStrings`
5.  运行`Deploy.ps1`
6.  配置 iis 网站
7.  运行`PostDeploy.ps1`

*(有关这些步骤的详细信息，请参见我们在 [XML 配置](http://octopusdeploy.com/documentation/features/xml-config)、 [PowerShell 脚本支持](http://octopusdeploy.com/documentation/features/powershell)和 [IIS 网站](http://octopusdeploy.com/documentation/features/iis)上的页面)*

### 部署失败. ps1

如果第 2-7 步失败，那么现在触手将寻找一个`DeployFailed.ps1`脚本，并调用它。它将可以访问其他 PowerShell 脚本获得的相同的[变量](http://octopusdeploy.com/documentation/features/variables)。这是一个放置恢复操作的好地方。

请注意，如果步骤 1 失败，则`DeployFailed.ps1` **不会被调用**。这有几个原因:

1.  `DeployFailed.ps1`无论如何也不可能被提取
2.  它所依赖的文件可能没有被提取
3.  在安装包之前，触手会检查可用的磁盘空间，所以这种情况应该很少发生
4.  解压软件包是一项独立的任务，所以首先没有什么要恢复的

一个很好的特性是传递最后安装的包的路径，恢复脚本可以在回滚时使用它。触须还没有足够的信息来做到这一点，但当[自动清除触须](https://trello.com/card/auto-purge-tentacles/4e907de70880ba000079b75c/20)实现时，它会做到，因为两者都需要保存一份以前安装的应用程序的列表。