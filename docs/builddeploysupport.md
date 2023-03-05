# BuildDeploySupport:在 Octopus - Octopus Deploy 中共享 PowerShell 脚本

> 原文：<https://octopus.com/blog/builddeploysupport>

在 Octopus 中有多个项目需要在部署期间完成相同的任务是很常见的，比如配置 IIS。复制和粘贴脚本当然是一种方法，但并不理想。在这篇客座博文中， **[Jonathan Goldman](https://twitter.com/jonnii)** 向我们展示了他在 Octopus 项目中分享 PowerShell 脚本库的技巧。

* * *

最近，我们已经将越来越多的项目转移到 Octopus Deploy，其中一件很快变得明显的事情是，需要在多个项目之间共享相同的部署脚本，例如安装服务、设置 windows 时间表或配置应用程序池。当脚本在项目之间复制和粘贴时，事情会变得很快变得混乱，所以我们寻找一个简单的解决方案。

为了解决这个问题，我创建了一个内部的 nuget 包，它包含了我们的部署脚本和一个 Init.ps1 步骤，将它们复制到一个解决方案文件夹中。然后，我们可以将脚本添加到需要它们的项目中，并从我们的 Octopus Deploy.ps1 脚本中引用它们。当一个脚本被更改时，我可以直接更新包并获得所有最新的更改。

因为我们创建的许多脚本最终都非常通用，所以我创建了一个每个人都可以利用的 nuget 包:

```
install-package BuildDeploySupport 
```

如果将 DeployWeb.ps1 作为链接添加到 Web 项目中(添加现有项->添加为链接)。并编辑您的 Deploy.ps1，如下所示

```
. .\DeployWeb.ps1

InstallAppPool 'my-app-pool' 'v4.0' {
    SetCredentials 'username' 'password'
}

InstallWebSite $OctopusWebSiteName 'my-app-pool' 'www.yourdomain.com' {
        SetWindowsAuthentication $true
        SetAnonymousAuthentication $false        
} 
```

您将不再需要登录服务器来配置应用程序池或创建网站。一切都应该正常。这类似于 Octopus 文档中概述的方法。

这不仅有助于人们开始使用 Octopus，还有助于围绕创建可重用的部署脚本培养社区。

如果你有兴趣投稿，github 上的所有内容都是开源的:

[https://github.com/jonnii/BuildDeploySupport](https://github.com/jonnii/BuildDeploySupport)