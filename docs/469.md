# 用八达通实现自动化。客户端- Octopus 部署

> 原文：<https://octopus.com/blog/octopus-client-automation>

在 Octopus Deploy 中，我们一直在喋喋不休地谈论 Octopus 是如何首先成为 API 的。作为 API first 的好处之一是，你可以通过 web 请求做 Octopus UI 做的任何事情。我们有一个名为 [Octopus 的便利图书馆。客户端](http://docs.octopusdeploy.com/display/OD/Octopus.Client)封装了 API，使得在 Octopus Deploy 服务器上执行操作变得容易。本文将重点介绍使用 PowerShell 的一些常见操作。

### 入门指南

有很多方法可以得到 Octopus.Client。有一个 [NuGet 包](https://www.nuget.org/packages/Octopus.Client/)可以包含在你的 Visual Studio 项目和 Octopus 中。客户端可以从 Octopus 服务器和触手安装文件夹中获得。

要开始使用 PowerShell，您需要参考 Octopus。客户端 dll:

```
Add-Type -Path 'Octopus.Client.dll' 
```

要在 Octopus 服务器上执行操作，您需要使用 [API 密钥](http://docs.octopusdeploy.com/display/OD/How+to+create+an+API+key)创建一个到 Octopus 服务器的连接:

```
$octopusURI = "http://localhost"
$apiKey = 'API-ALLYOURBASEBELONGTOUS'
$endpoint = new-object Octopus.Client.OctopusServerEndpoint $octopusURI, $apiKey 
```

最后，使用连接访问 Octopus 服务器存储库:

```
$repository = new-object Octopus.Client.OctopusRepository $endpoint 
```

### 简单的操作

你的八达通服务器上的大部分资源都可以通过八达通访问。客户端通过一组常见的操作:

*   FindAll()
*   获取(id)
*   创建(资源)
*   修改(资源)
*   删除(资源)

例如，要创建一个环境:

```
Add-Type -Path 'Octopus.Client.dll'

$octopusURI = "http://localhost"
$apiKey = 'API-ALLYOURBASEBELONGTOUS'

$endpoint = new-object Octopus.Client.OctopusServerEndpoint $octopusURI, $apiKey 
$repository = new-object Octopus.Client.OctopusRepository $endpoint

$environment = new-object Octopus.Client.Model.EnvironmentResource
$environment.Name = "Demo environment"
$environment.Description = "An environment for demonstrating Octopus.Client"

$repository.Environments.Create($environment) 
```

### 自动化所有的事情

章鱼的真正力量。客户端是自动化的能力。例如:您可以提供一组触角，将它们添加到一个环境中，然后通过按 Octopus UI 中的一组按钮将所有项目的最新版本部署到该环境中，或者您可以编写一个脚本。

向环境中添加监听触角:

```
Add-Type -Path 'Octopus.Client.dll'

$octopusURI = "http://localhost"
$apiKey = 'API-ALLYOURBASEBELONGTOUS'

$endpoint = new-object Octopus.Client.OctopusServerEndpoint $octopusURI, $apiKey 
$repository = new-object Octopus.Client.OctopusRepository $endpoint

$tentacleEndpoint = New-Object Octopus.Client.Model.EndPoints.ListeningTentacleEndpointResource
$tentacleEndpoint.Thumbprint = "B0EDD32958AC90743F21F96DD9AAD5E13AF202EF"
$tentacleEndpoint.Uri = "https://localhost:10933"

$environment = $repository.Environments.FindByName("Demo environment")

$tentacle = New-Object Octopus.Client.Model.MachineResource
$tentacle.Endpoint = $tentacleEndpoint
$tentacle.EnvironmentIds.Add($environment.Id)
$tentacle.Roles.Add("demo-role")
$tentacle.Name = "Demo Tentacle"

$repository.Machines.Create($tentacle) 
```

将每个项目的版本部署到环境中:

```
Add-Type -Path 'Octopus.Client.dll'

$octopusURI = "http://localhost"
$apiKey = 'API-ALLYOURBASEBELONGTOUS'

$endpoint = new-object Octopus.Client.OctopusServerEndpoint $octopusURI, $apiKey 
$repository = new-object Octopus.Client.OctopusRepository $endpoint

$environment = $repository.Environments.FindByName("Demo environment")

$projects = $repository.Projects.FindAll()
foreach ($project in $projects)
{
    $releases = $repository.Projects.GetReleases($project)
    $latestRelease = $releases.Items | Select-Object -first 1
    $deployment = new-object Octopus.Client.Model.DeploymentResource
    $deployment.ReleaseId = $latestRelease.Id
    $deployment.EnvironmentId = $environment.Id
    $repository.Deployments.Create($deployment)
} 
```

通过一点创造性，您可以自动化您的部署自动化系统，并与您喜欢的工具集成。

### 解决纷争

Octopus Server API 是一个很好的起点，可以帮助您了解更多关于可用资源以及它们是如何构成的。在您的 Octopus 服务器上浏览到`/api`即可获得。在通过 web 界面使用 Octopus Server 的同时观看 [Fiddler](http://www.telerik.com/fiddler) 可以帮助理解更复杂的资源交互，比如创建变量集和部署过程。最后，我发现探索 Visual Studio 和 PowerShell ISE 中的智能感知有助于熟悉 Octopus 中可用的方法和属性。客户端(通过扩展，还有 API)。

快乐的自动化！