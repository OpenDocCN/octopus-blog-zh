# 将 ASP.NET 应用程序部署到 Azure 网站- Octopus Deploy

> 原文：<https://octopus.com/blog/deploy-aspnet-applications-to-azure-websites>

*现在回到最初的博文...*

最近，越来越多的人希望从 Octopus 部署他们的 Azure 网站。问题是目前还没有这方面的 OOTB 功能。

由于目前没有内置的方法来完成它，我们的一个用户创建了一个 step 模板(它可以在 [Octopus Library](http://library.octopusdeploy.com) 网站上找到，还有一堆其他有用的 step 模板),它运行一个 PowerShell 脚本，该脚本使用 Web Deploy 将您的应用程序部署到 Azure，考虑到这一点，我想我应该写一个小的(有点)博客帖子，逐步介绍如何使用这个 step 模板设置您的 ASP.NET 应用程序并准备好部署到 Azure。

出于这篇博文的目的，我将在 Visual Studio 2013 中创建一个演示 ASP.NET MVC 应用程序。

### 创建您的 ASP.NET 应用程序

首先让我们选择我们的项目类型，并给它一个名字

![Create New Project](img/5db38595607c1c42facfd95d4ab2701a.png)

然后指定要使用的模板，我将只使用提供的 MVC 模板，我将保留`Host in the cloud`复选框未选中，因为我想使用 Octopus Deploy 来处理我的部署。

![Specify Web Template](img/9f0fbc67bd30fb58aeedd9f5048bf00a.png)

一旦项目被创建，按 F5 运行你的新的和闪亮的 ASP.NET MVC 应用程序。

![Web site up and running](img/75cc317331b63926334fb7eb07c422f6.png)

没什么太激动人心的，但是它给了我们一个起点，我们可以从这里开始部署设置和运行。

#### 创建 NuGet 包

由于 Octopus Deploy 在部署您的应用程序时使用 NuGet 包，我们创建了一个小工具，它将从您构建项目时创建的输出文件中创建一个 NuGet 包。

***边注**:OctoPack 创建的 NuGet 包和你从 NuGet 图库安装的 NuGet 包略有不同。我们的 NuGet 包只是一堆文件和文件夹，它们组成了你的应用程序的结构。*

##### 将 OctoPack NuGet 包添加到您的项目中

要将 OctoPack 添加到我们的项目中，右键单击您的解决方案并选择 Manage NuGet Packages for Solution，搜索`octopack`并单击`Install`按钮。

![Install OctoPack](img/04502296833835b03ab67c5b8185ebd5.png)

选择要安装 OctoPack 的项目，在我的例子中，我只有一个项目，所以我选择它并单击确定。

![Select project to install OctoPack in](img/3caa946a4be8db50fe333443aee65fa6.png)

![OctoPack Installed](img/6502108f6ccafd3e27d447786d690d31.png)

###### 使用 MSBuild 从命令行构建项目并生成 NuGet 包

现在 OctoPack 已经安装好了，当我们从命令行构建我们的解决方案时，我们可以告诉它为我们生成一个 NuGet 包。

在命令提示符下输入:

```
C:\Code\OctoWeb\OctoWeb>msbuild OctoWeb.sln /t:build /p:RunOctoPack=true 
```

如果一切正常，您应该会看到类似下面的输出:

```
Microsoft (R) Build Engine version 12.0.30723.0
[Microsoft .NET Framework, version 4.0.30319.34014]
Copyright (C) Microsoft Corporation. All rights reserved.

Building the projects in this solution one at a time. To enable parallel build, please add the "
/m" switch.
Build started 23/09/2014 3:25:24 PM.
Project "C:\Code\OctoWeb\OctoWeb\OctoWeb.sln" on node 1 (build target(s)).
ValidateSolutionConfiguration:
  Building solution configuration "Debug|Any CPU".
Project "C:\Code\OctoWeb\OctoWeb\OctoWeb.sln" (1) is building "C:\Code\OctoWeb\OctoWeb\OctoWeb\
OctoWeb.csproj" (2) on node 1 (default targets).
...
CopyFilesToOutputDirectory:
  OctoWeb -> C:\Code\OctoWeb\OctoWeb\OctoWeb\bin\OctoWeb.dll
OctoPack:
  OctoPack: Get version info from assembly: C:\Code\OctoWeb\OctoWeb\OctoWeb\bin\OctoWeb.dll
  Using package version: 1.0.0.0
  OctoPack: Written files: 101
  OctoPack: A NuSpec file named 'OctoWeb.nuspec' was not found in the project root, so the file
   will be generated automatically. However, you should consider creating your own NuSpec file  
  so that you can customize the description properly.
  OctoPack: Packaging an ASP.NET web application
  OctoPack: Add content files
  ...
  OctoPack: Add binary files to the bin folder
  ...
  OctoPack: Attempting to build package from 'OctoWeb.nuspec'.
  OctoPack: Successfully created package 'C:\Code\OctoWeb\OctoWeb\OctoWeb\obj\octopacked\OctoWe
  b.1.0.0.0.nupkg'.
  OctoPack: Copy file: C:\Code\OctoWeb\OctoWeb\OctoWeb\obj\octopacked\OctoWeb.1.0.0.0.nupkg
  OctoPack: OctoPack successful
Done Building Project "C:\Code\OctoWeb\OctoWeb\OctoWeb\OctoWeb.csproj" (default targets).

Done Building Project "C:\Code\OctoWeb\OctoWeb\OctoWeb.sln" (build target(s)).

Build succeeded.
    0 Warning(s)
    0 Error(s)

Time Elapsed 00:00:01.86 
```

如果你看一下 OctoPack 说它创建了 NuGet 包的文件夹，你应该看到它确实在那里。

![OctoPack NuGet package created](img/887881ccdbab83340c9b5091cbfdac9f.png)

如果您在 NuGet Package Explorer 中打开生成的 NuGet 包，您应该看到 OctoPack 已经打包了您的网站，因为它将被部署到您的 web 服务器。

![NuGet Package Explorer](img/b1af8cbf4fd5d42aa04c20b2e2e9c446.png)

如果您想让 OctoPack 将创建的 NuGet 包复制到本地文件夹或文件共享，您可以使用下面的调用`msbuild`

```
C:\Code\OctoWeb\OctoWeb>msbuild OctoWeb.sln /t:build /p:RunOctoPack=true /p:OctoPackPublishPackageToFileShare=C:\NuGet 
```

或者，发布到 Octopus 中的内置存储库中

```
C:\Code\OctoWeb\OctoWeb>msbuild OctoWeb.sln /t:build /p:RunOctoPack=true /p:OctoPackPublishPackageToHttp=http://your-octopus-server/nuget/packages /p:OctoPackPublishApiKey=API-ABCDEFGMYAPIKEY 
```

###### 修改。csproj 在构建项目时生成 NuGet 包

如果您想在每次构建解决方案时生成 NuGet 包并将其发布到本地文件共享，那么您可以修改您的`.csproj`文件并将以下 OctoPack 标记添加到项目属性组中:

```
<PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    ...
    <RunOctoPack>true</RunOctoPack>
    <OctoPackPublishPackageToFileShare>C:\Packages</OctoPackPublishPackageToFileShare>
</PropertyGroup> 
```

或者，如果您希望在执行调试构建时将 NuGet 包发布到本地文件共享，并且仅在执行发布构建时发布到内置存储库:

```
<PropertyGroup>
    <Configuration Condition=" '$(Configuration)' == '' ">Debug</Configuration>
    <Platform Condition=" '$(Platform)' == '' ">AnyCPU</Platform>
    ...
    <RunOctoPack>true</RunOctoPack>
</PropertyGroup>
<PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Debug|AnyCPU' ">
    <OctoPackPublishPackageToFileShare>C:\Packages</OctoPackPublishPackageToFileShare>
</PropertyGroup>
<PropertyGroup Condition=" '$(Configuration)|$(Platform)' == 'Release|AnyCPU' ">
    ...
    <OctoPackPublishPackageToHttp>http://your-octopus-server/nuget/packages</OctoPackPublishPackageToHttp>
    <OctoPackPublishApiKey>API-ABCDEFGMYAPIKEY</OctoPackPublishApiKey>
</PropertyGroup> 
```

* * *

### 设置您的新 Azure 网站

现在我们将设置我们将要部署到的 Azure 网站。

登录到 [Azure 管理门户](https://manage.windowsazure.com)并创建一个新网站。

![Create New Azure Web Site](img/b8be4d8509390640d7a7f7f2e39cb4f4.png)

一旦创建了网站，

![Azure Web Site created](img/e987479ed4a76c188ea80cdb615e2ec2.png)

单击它可以访问网站的设置。

#### 下载发布配置文件

从起始页，我们将下载发布配置文件设置文件，以获取在 Octopus Deploy 中设置部署流程所需的值。所以点击`Download the publish profile`链接下载所需的设置文件。

![Download Publish Profile](img/5fc2fe561e2360af65f5541db0931c86.png)

现在我们已经获得了 Octopus 部署设置之外的所有东西，我们可以继续到您的 Octopus 服务器来获得我们的项目和部署过程设置，并准备好部署您的新 Azure 网站。

* * *

### 将 Web 部署步骤模板添加到 Octopus

我们需要做的第一件事是，现在我们已经将我们的网站打包到一个 NuGet 包中，从 [Octopus 库站点](http://library.octopusdeploy.com)导入`Web Deploy - Publish Website (MSDeploy)`步骤模板。

#### 从 Octopus 库中获取“Web 部署-发布网站(MSDeploy)”步骤模板

在 Octopus 库站点上，搜索 web deploy

![Web Deploy Step Template](img/163e89086a29de7a5494e0842a3396d1.png)

单击返回的结果，然后单击绿色的大按钮`Copy to clipboard`。

![Web Deploy Step Template details](img/d45035cdba274ea46d68810add95d12a.png)

#### 将步骤模板导入 Octopus Deploy

登录到你的八达通服务器，进入图书馆->步骤模板。在步骤模板选项卡上，单击`Import`链接

![Import Step Template](img/7f42cfaaacde1015e056b3e4cb5065dd.png)

这将显示`Import`对话框，将您从 Octopus 库站点复制的 Web Deploy 步骤模板粘贴到提供的文本区域中。

![Import](img/1b73efff4d655350133fc3d663533c96.png)

点击`Import`按钮，步骤模板将被导入

![Imported](img/73fb68809713504380008c2e772b4336.png)

很好，现在我们已经准备好设置我们的项目和部署流程，开始将我们的 ASP.NET MVC 应用程序部署到 Azure 网站。

* * *

### 在 Octopus Deploy 中设置项目

接下来要做的是设置我们的新项目，并定义部署流程，我们将使用该流程将我们的 ASP.NET MVC 应用程序部署到我们的 Azure 网站。

#### 创建我们的项目

进入“项目”->“所有项目”，然后点击项目组上的`Add Project`按钮。

![Create New Octopus Project](img/981cff4330ef2b8d3f854f9a05aee400.png)

![Create New Octopus Project Details](img/4872bcf9df7818a54c7d255aef8ee5ee.png)

![Octopus Project](img/753ea49e1c0e9ee1b65fba044c5c3e8e.png)

恭喜你，你得到了一个闪亮的新项目！；)

#### 定义您的项目变量

为了使设置您的项目来部署多个环境变得容易，我们将创建可以限定环境、角色和机器范围的项目变量。

打开项目站点上的变量选项卡，然后为网站名称、发布 URL、用户名和密码添加变量。我们将把密码变量设为`Sensitive Variable`,这样我们就可以保密。

打开您先前下载的发布概要文件，并从 Web Deploy 的发布概要文件中获取`publishUrl`、`msdeploySite`、`userName`和`userPWD`(例如`<publishProfile profileName="octowebdemo - Web Deploy">`，然后在适当的变量中填入值。然后点击`Save`。

![Octopus Project Variables](img/046a3d9d547fc6c2b50e1fa16dcb73c8.png)

在这个演示中，我不会限定变量的范围，因为我只有一个环境、一台机器和一个角色。

#### 定义您的部署流程

好了，现在我们来谈谈整个过程的商业方面。

现在我们开始为我们的项目指定部署过程，它将由两个步骤组成，一个 NuGet 包步骤和我们为 Web Deploy 导入的 PowerShell 步骤。

***可选地*** 一旦部署完成，您可以添加另一个步骤来“预热”网站。恰好在图书馆网站上，我们有一个步骤模板。搜索`Test URL`并导入返回的步骤模板。

打开项目页面上的`Process`选项卡。

![Deployment Process tab](img/6d20063d630e233456182b19ae707f55.png)

##### 添加“部署 NuGet 包”步骤

点击`Add Step`按钮。

选择`Deploy a NuGet package`步骤。

![NuGet Package step](img/3be042b29d22bd88a330bc6ba5c3ad3f.png)

![NuGet Package step setup](img/2ab265e63b59225f7c909f89ed569c02.png)

填写必要的细节，从发布它的 NuGet 提要(在我的例子中是磁盘上的本地文件夹)中指定您的 web 应用程序 NuGet 包。

![NuGet Package step completed](img/b8369fb44cdc6fd7f65c49dbe9303d1e.png)

点击`Save`

![Project deployment process with 1 step](img/3f91e4c87d2d4dbd4d45a24a3fa5ea85.png)

##### 添加“Web 部署-发布网站(MSDeploy)”步骤

现在是时候添加我们的 Web 部署步骤了，再次点击`Add Step`按钮并选择`Web Deploy - Publish Website (MSDeploy)`

![Web Deploy step](img/62c78b79d5a9099f6e843aa611b0b8b9.png)

填写必要的细节，使用变量绑定来指定 Azure 特定的细节。

![Web Deploy step details completed](img/7ea1e7e7ea0d06e82670c67e730ec024.png)

所需的`Package Step Name`是提取 NuGet 包中包含的文件的步骤的名称，这用于在磁盘上定位需要上传到 Azure 网站的文件。

点击`Save`

![Project deployment process with 2 steps](img/d06366abdb547b8a58922e4ef5f59467.png)

就这样，我们现在准备创建一个版本并将其部署到 Azure。

### 创建一个版本

要创建一个发布，点击项目页面顶部的`Create Release`按钮。

![Create a Release](img/313726e9221dcf11b3d39ae8a4d7b59c.png)

在 release 页面上，您可以选择为 Octopus Deploy 将为您预先填充的内容指定一个不同的版本号(基于您选择的项目设置)，为您的 web 应用程序指定要使用的 NuGet 包的版本号(我只有 1 个版本，所以我将使用它)以及发行版中应该包含的任何发行说明。

![Release details completed](img/0f1baa7226a95520c14c8af6eb7e7c6a.png)

点击`Save`。这将带您进入发布概述页面。

![Release Overview](img/7cca94e712aebc8e22d674127d76ed89.png)

现在我们想把这个版本部署到我们的 Azure 网站上，所以点击`Deploy this release`按钮。选择要部署到的环境。在我的例子中，我只有我的`Dev`环境设置，所以我将选择这个。

![Deploy Release to Dev](img/060048d23e1f03805942d67b7a82b756.png)

![Deploy Release to Dev details](img/728e168482cbc8fa2e2342f1de24184c.png)

在 deployment 页面上，您可以选择将发布安排在稍后的日期/时间，以及部署哪些步骤(以及部署到什么机器)。

我将坚持使用默认设置，只需点击`Deploy Release`按钮。

![Deploy progressed](img/b4293456cb22dcdc7625a86ded17b6f8.png)

部署完成后，打开一个浏览器，然后浏览 Azure 中运行的 web 应用程序的 URL。

![Deploy completed](img/2a990f2a6f331a7850ef2e2b0bd3d604.png)

![Azure Web Site running](img/17e358357bb9e8dfeca90313f9916c03.png)

恭喜，您已经使用 Web Deploy 从 Octopus Deploy 部署了您的 Azure 网站！

* * *

### 将应用程序更新并重新部署到 Azure

现在我们已经将应用程序从 Octopus 部署到 Azure，让我们对应用程序做一些修改，然后将新版本部署到 Azure。

我将更新应用程序的名称，以及使用的一些颜色。

![Modified web site running](img/d937d927ce0002ee91d440dd6b5e817a.png)

#### 重新创建您的 NuGet 包

当从命令行重新创建 NuGet 包时，不使用存储在`AssemblyInfo.cs`文件的`[assembly: AssemblyVersion]`中的版本，我将通过将`OctoPackPackageVersion`参数传递给 MSBuild 来覆盖它。

```
C:\Code\OctoWeb\OctoWeb>msbuild OctoWeb.sln /t:build /p:RunOctoPack=true /p:OctoPackPackageVersion=1.0.0.1 
```

最终结果应该类似于下图

```
 OctoPack: Attempting to build package from 'OctoWeb.nuspec'.
  OctoPack: Successfully created package 'C:\Code\OctoWeb\OctoWeb\OctoWeb\obj\octopacked\OctoWeb.1.0.0.1.nupkg'. 
```

将新的 NuGet 包复制到 Octopus 可以访问的位置。

#### 在 Octopus Deploy 中创建新版本

在 Octopus 中，返回到发布选项卡并点击`Create Release`。Octopus 现在应该拿起最新的包(v1.0.0.1)。

![Create a new release](img/5c2d4d4fb266b9d9750bd58a98e69212.png)

点击`Save`。

#### 将最新版本部署到 Azure 网站

现在剩下的就是将新版本(`0.0.2`)部署到 Azure。

点击`Deploy this release`，选择要部署到的环境，最后点击`Deploy Release`。

![Create new release completed](img/812fbba636826f2a95a87f57ccbfc8fd.png)

![Deploy new release in progress](img/a740c65cac96f3f04f36d511285d6e53.png)

此部署应该比初始部署快得多，因为它将只上传已更改的文件。部署完成后，Azure 网站应该会根据所做的更改进行更新。

![Deploy new release completed](img/7f291770d10dedddaeb05fe74e3a8fa9.png)

![Modified Azure Web Site running](img/e306f7ba3bf3813d4f0445935dfa3d32.png)

* * *