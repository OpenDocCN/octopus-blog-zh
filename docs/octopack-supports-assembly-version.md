# OctoPack 现在支持汇编版本- Octopus Deploy

> 原文：<https://octopus.com/blog/octopack-supports-assembly-version>

当使用 [OctoPack](https://github.com/OctopusDeploy/OctoPack "OctoPack on GitHub") 时，构建的 NuGet 包将默认为版本 1.0.0。这可以通过传递以下命令从 MSBuild 命令行重写:

```
/p:OctopusPackageVersion=2.0.0 
```

这在使用生成版本号的构建工具时工作得很好，因为您通常可以将参数传递给 MSBuild。

前段时间在 GitHub 上开了一个问题来增加对此的支持:

```
[assembly: AssemblyVersion("1.0.*")] 
```

我不知道不添加自定义任务来解决这个问题的方法，但是[安德烈](https://github.com/andrebires "André")发布了一个使用[getassemblyideentity](http://msdn.microsoft.com/en-us/library/ms164296.aspx "GetAssemblyIdentity")MSBuild 任务的例子。感谢 André，OctoPack 1.0.99 现在将从主输出程序集读取版本号，如果它没有被从命令行传递的`/p:OctopusPackageVersion`覆盖的话。

你可以在 GitHub 上[查看差异](https://github.com/OctopusDeploy/OctoPack/commit/83a65458478a9bbf8807b27ab464d7fe24f0387c "Diff")。