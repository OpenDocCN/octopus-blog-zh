# OctoPack 2.0 - Octopus 部署

> 原文：<https://octopus.com/blog/octopack-improvements>

当开始使用 Octopus Deploy 时，人们经常会在最初的步骤中遇到困难:将他们的应用程序打包成 NuGet 包。为了更容易地创建包，我们在我们的 NuGet 打包助手 OctoPack 中实现了一些大的改变，创建了[octo pack 2.0 版](https://github.com/OctopusDeploy/OctoPack/)。

最大的变化是我们现在让调用 OctoPack 变得更加容易和简单。您只需运行:

```
msbuild YourSolution.sln /t:Build /p:RunOctoPack=true 
```

OctoPack 将只使用相关文件自动打包您的应用程序(例如，web 应用程序的内容文件和二进制文件)。这是对 OctoPack 1.0 的一个很大的改进，因为你不再需要独立地构建每个项目(我们正在构建上面的解决方案)，并且**你不再需要处理 web 应用程序项目发布**了——octo pack 会处理它。

我们还实现了其他一些改进:

1.  您不再需要创建 NuSpec 文件——如果不存在，我们会自动生成它
2.  您可以从我们打包时包含的文件中添加发行说明
3.  卸载时，我们现在删除目标文件

有关 OctoPack 2.0 的更多信息，请[查看自述文件](https://github.com/OctopusDeploy/OctoPack/blob/master/readme.md)。我将在博客上发布一些如何在团队城市和 TFS 中使用 OctoPack 2.0 的例子，但在此之前，我希望这些变化能让你更容易地为 Octopus 打包应用程序。愉快的部署！