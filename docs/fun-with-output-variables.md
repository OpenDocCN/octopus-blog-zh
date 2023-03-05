# 有趣的输出变量-章鱼部署

> 原文：<https://octopus.com/blog/fun-with-output-variables>

在 Octopus 2.4 中，我们增加了一个步骤中的变量在另一个步骤中可用的能力。例如，您可能有一个名为 **StepA** 的独立 PowerShell 步骤，它的功能如下:

```
Set-OctopusVariable -name "TestResult" -value "Passed" 
```

然后，您可以在后续部署步骤(在同一个部署中)中使用它，如下所示:

```
$TestResult = $OctopusParameters["Octopus.Action[StepA].Output.TestResult"] 
```

## 内置输出变量

在一个步骤运行之后，Octopus 捕获输出变量，并保存它们以供后续步骤使用。除了您自己使用`Set-OctopusVariable`创建的变量，Octopus 还提供了许多内置变量:

*   对于 NuGet 包步骤:
    *   `Octopus.Action[StepName].Output.Package.InstallationDirectoryPath`:包部署到的路径。
*   对于手动干预步骤:
    *   `Octopus.Action[StepName].Output.Manual.Notes`:响应手动步骤输入的注释。
    *   `Octopus.Action[StepName].Output.Manual.ResponsibleUser.Id`
    *   `Octopus.Action[StepName].Output.Manual.ResponsibleUser.Username`
    *   `Octopus.Action[StepName].Output.Manual.ResponsibleUser.DisplayName`
    *   `Octopus.Action[StepName].Output.Manual.ResponsibleUser.EmailAddress`

## 输出变量的范围和索引

重要的是要记住，与构建服务器不同，Octopus 在许多机器上并行运行各个步骤。这意味着每台机器可能对相同的输出变量产生不同的值。

例如，假设我们有两台机器， **App01** 和 **App02** ，我们在这两台机器上运行这个脚本:

```
Set-OctopusVariable -name "MyMachineName" -value [System.Environment]::MachineName 
```

显然，我们将有两个不同的值可用，因为两台机器有不同的主机名。为了处理这一点，Octopus 创建了变量，并将它们限定在一台机器上。在这个例子中，Octopus 将存储两个变量:

```
Octopus.Action[StepA].Output.MyMachineName = App01     (Scope: App01)  # Value from App01 machine
Octopus.Action[StepA].Output.MyMachineName = App02     (Scope: App02)  # Value from App02 machine 
```

从现在开始，在这些机器上运行的部署中的任何步骤都将从该机器获得适用的输出变量。这意味着您可以:

```
$name = $OctopusParameters["Octopus.Action[StepA].Output.MyMachineName"] 
```

有时，您可能需要从一台机器上访问由另一台机器产生的变量。在这种情况下，Octopus 还存储*非作用域*变量，这些变量由机器用*索引*:

```
Octopus.Action[StepA].Output[App01].MyMachineName = App01              # Value from App01 machine
Octopus.Action[StepA].Output[App02].MyMachineName = App02              # Value from App02 machine 
```

这意味着，例如在 **App03** 上运行的后续步骤中，您可以:

```
$app01Name = $OctopusParameters["Octopus.Action[StepA].Output[App01].MyMachineName"]
$app02Name = $OctopusParameters["Octopus.Action[StepA].Output[App02].MyMachineName"]
# Do something with $app01Name and $app02Name 
```

记住`$OctopusParameters`只是一个`Dictionary<string,string>`。这意味着你可以这样做:

```
$MatchRegex = "Octopus\.Action\[StepA\]\.Output\[(.*?)\]\.MyMachineName"

Write-Host "Machine names:"
$OctopusParameters.GetEnumerator() | Where-Object { $_.Key -match $MatchRegex } | % { 
  Write-Host "$_.Value"
} 
```

这里，我们迭代字典中的所有键/值对，并找到与我们的 regex 匹配的键/值对，regex 在变量键的机器名组件上有一个通配符。

## 查找先前软件包的安装位置

对于输出变量来说，这是一个如此常见的用例，以至于我想显式地调用它。

默认情况下，为了避免各种问题，破坏部署和文件锁，触须自动提取包到一个新的，干净的目录。如果您多次部署完全相同的包，您将会得到类似如下的结果:

```
C:\Octopus\Applications\Production\MyApp\1.0.0
C:\Octopus\Applications\Production\MyApp\1.0.0_1
C:\Octopus\Applications\Production\MyApp\1.0.0_2
C:\Octopus\Applications\Production\MyApp\1.0.0_3 
```

假设您部署了一个 NuGet 包，但是想要编写一个独立的 PowerShell 脚本，该脚本在包被提取到的目录中的同一服务器上运行，但是不是 NuGet 包的一部分。你可以用这个:

```
$packageDir = $OctopusParameters["Octopus.Action[MyApp].Output.Package.InstallationDirectoryPath"]
cd $packageDir

# Do your custom logic 
```

## 摘要

应用程序部署通常涉及在许多不同的机器上运行部署包和执行代码。Octopus 中的输出变量提供了一种非常强大的方式来在不同的步骤和不同的机器之间共享这些值。我希望你会发现这个功能很有用！

### 了解更多信息