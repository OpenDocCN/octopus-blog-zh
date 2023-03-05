# 使用 DbUp 和 Octopus workers 实现数据库部署自动化- Octopus Deploy

> 原文：<https://octopus.com/blog/dbup-database-deployments>

[![Using DbUp and Octopus workers for database deployment automation](img/2b0b02ea2211b6525b3a731554b0ebb7.png)](#)

在过去十年中，数据库部署最令人兴奋的一个方面是已经发布的工具数量。看一下我以前关于这个话题的帖子，你会发现我明显偏向于 Redgate 的工具，但是我是 Redgate 的朋友，这是有原因的。

在这个帖子里，我用的是 [DbUp](https://dbup.readthedocs.io/en/latest/) 。DbUp 是一个免费的开源工具，我们在 Octopus Deploy 中使用它进行数据库部署。每当您安装或升级 Octopus Deploy 时，DbUp 都会运行脚本来更新您的数据库。我们的创始人 Paul Stovell 在 2012 年写了一篇关于如何使用 DbUp 部署到 SQL Server 的[博客文章](https://octopus.com/blog/howto/deploy-a-sql-database)。在很大程度上，那篇博文至今仍然有效。

这篇文章是那篇旧文章的更新。DbUp 和 Octopus Deploy 中添加了许多新特性，我将介绍其中的一些特性，并创建一个过程来使用它进行数据库部署。它甚至包括一个 DBA 批准的审查步骤。

## 对 DbUp 的更改

本质上，DbUp 是一个脚本运行器。对数据库的更改是通过脚本完成的:

*   Script001_AddTableA.sql
*   script 002 _ addcolumntesttotablea . SQL
*   script 003 _ addcolumnstategaintotablea . SQL

DbUp 通过您自己编写的控制台应用程序运行，因此您可以控制使用哪些选项，并且不需要大量代码:

```
static int Main(string[] args)
{
    var connectionString =
        args.FirstOrDefault()
        ?? "Server=(local)\\SqlExpress; Database=MyApp; Trusted_connection=true";

    var upgrader =
        DeployChanges.To
            .SqlDatabase(connectionString)
            .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly())
            .LogToConsole()
            .Build();

    var result = upgrader.PerformUpgrade();

    if (!result.Successful)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine(result.Error);
        Console.ResetColor();

        return -1;
        }
    }

    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine("Success!");
    Console.ResetColor();
    return 0;
} 
```

您捆绑这些脚本并告诉 DbUp 运行它们。它将该列表与存储在目标数据库中的列表进行比较。将运行不在该目标数据库列表中的任何脚本。脚本按字母顺序执行，每个脚本的结果都显示在控制台上。非常容易实现和理解。

[![](img/5a69c60e9e32aaf2cfa55f98507a3f3b.png)](#)

当您部署到开发或测试环境时，这非常有用。我交谈过的许多公司更喜欢他们的 DBA 在投入生产之前批准脚本。也可能是一个试运行或预生产环境。这个批准过程是必不可少的，尤其是当您第一次开始部署数据库时。

### HTML 报告

迁移脚本是一把双刃剑，就像 C++中的内存管理一样。你拥有完全的控制权，这给了你巨大的力量。但是，也很容易搞砸。这完全取决于所做更改的类型和作者的 SQL 技能。当没有经验的 C#开发人员编写这些迁移脚本时，DBA 对这个过程的信任度会很低。

最近，DbUp 增加了生成 HTML 报告的功能。这是一个扩展方法，您可以给它您想要生成的报告的路径。这意味着这部分从:

```
var result = upgrader.PerformUpgrade();

if (!result.Successful)
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine(result.Error);
    Console.ResetColor();

    return -1;
    }
} 
```

收件人:

```
// --generateReport is the name of the example argument.  You can call it anything
if (args.Any(a => "--generateReport".Equals(a, StringComparison.InvariantCultureIgnoreCase)))
{
    upgrader.GenerateUpgradeHtmlReport("C:\\DeploymentLocation\\UpgradeReport.html");
}
else
{
    var result = upgrader.PerformUpgrade();

    if (!result.Successful)
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine(result.Error);
        Console.ResetColor();
        return -1;
    }
} 
```

该代码将生成一个包含所有将要运行的脚本的报告。

[![](img/d56514026cbbdc1d07eb54245c97728e.png)](#)

### 始终运行脚本和脚本分组

默认情况下，DbUp 将运行一次脚本，大多数情况下这没问题，但有时总是运行一个脚本或一组脚本也不错。一个例子是刷新所有视图的部署后脚本。或者，使用一个脚本来重建所有索引并重新生成统计数据。您不希望为每个部署编写新的脚本。

DbUp 最近增加的另一个特性是能够将一组脚本标记为`AlwaysRun`并提供一个运行组:

```
var upgradeEngineBuilder = DeployChanges.To
    .SqlDatabase(connectionString, null) //null or "" for default schema for user
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), script => script.StartsWith("SampleApplication.PreDeployment."), new SqlScriptOptions { ScriptType = ScriptType.RunAlways, RunGroupOrder = 1})
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), script => script.StartsWith("SampleApplication.Scripts."), new SqlScriptOptions { ScriptType = ScriptType.RunOnce, RunGroupOrder = 2})
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), script => script.StartsWith("SampleApplication.PostDeployment."), new SqlScriptOptions { ScriptType = ScriptType.RunAlways, RunGroupOrder = 3})
    .LogToConsole();

var upgrader = upgradeEngineBuilder.Build();

var result = upgrader.PerformUpgrade();

// Display the result
if (result.Successful)
{
    Console.ForegroundColor = ConsoleColor.Green;
    Console.WriteLine("Success!");
}
else
{
    Console.ForegroundColor = ConsoleColor.Red;
    Console.WriteLine(result.Error);
    Console.WriteLine("Failed!");
} 
```

## 创建 DbUp 控制台应用程序

有了这些新特性，我们将构建一个. NET 核心 DbUp 控制台应用程序来部署到 SQL Server。然后，我们将在 Octopus 部署中整合一个流程来运行控制台应用程序。

下面的所有代码都可以在 [GitHub repo](https://github.com/OctopusSamples/DbUpSample) 中找到。

我选择了。网核结束。NET 框架，因为它可以在任何地方构建和运行。DbUp 是一个. NET 标准库。DbUp 在. NET Framework 应用程序中也能很好地工作。

让我们启动我们选择的 IDE，创建一个. NET 核心控制台应用程序。我使用 JetBrain 的 Rider 来构建这个控制台应用程序。比起 Visual Studio 我更喜欢它。

### 脚手架

控制台应用程序已经创建。现在我们需要引入 DbUp NuGet 包。让我们转到我们的 NuGet 包管理器:

【T2 ![](img/f523d53a7bdac6d4dbd2be2c465858d9.png)

接下来，我们选择 DbUp-SqlServer 包。该包包括核心包以及部署到 SQL Server 的必要代码。如果您想部署到 PostgreSQL、MySQL、Oracle 或 SQLite，您可以选择:

[![](img/9aa6e79cc0649efee37d46c514aed97e.png)](#)

控制台应用程序需要一些脚本来部署。我将添加三个文件夹，并用一些脚本文件填充它们:

[![](img/7df6b3dcfc03738af8b838d5e2b6e662.png)](#)

建议你加个前缀，比如 001，002 等。，添加到脚本文件名的开头。DbUp 按字母顺序运行脚本，该前缀有助于确保脚本按正确的顺序运行。

默认情况下，。NET 在构建控制台应用程序时不会包含这些脚本文件，我们希望将这些脚本文件作为嵌入式资源包含在内。幸运的是，我们可以通过在`.csproj`文件中包含这段代码来轻松地添加对这些文件的引用:

```
 <ItemGroup>
        <EmbeddedResource Include="BeforeDeploymentScripts\*.sql" />
        <EmbeddedResource Include="DeploymentScripts\*.sql" />
        <EmbeddedResource Include="PostDeploymentScripts\*.sql" />
    </ItemGroup> 
```

整个文件如下所示:

[![](img/ad389496812acd55f951f5bc71ccac8c.png)](#)

### Program.cs 文件

启动这个应用程序的最后一步是在`Program.cs`中添加必要的代码来调用 DbUp。应用程序接受来自命令行的参数，Octopus Deploy 将被配置为发送以下参数:

*   **ConnectionString** :在这个演示中，我们将它作为参数发送，而不是存储在配置文件中。
*   **PreviewReportPath** :保存预览报表的完整路径。完整路径参数是可选的。当它被发送进来时，我们为 Octopus Deploy 生成一个预览 HTML 报告，以变成一个工件。当它没有被发送进来时，代码将执行实际的部署。

让我们从命令行参数中提取连接字符串开始:

```
static void Main(string[] args)
{    
    var connectionString = args.FirstOrDefault(x => x.StartsWith("--ConnectionString", StringComparison.OrdinalIgnoreCase));

    // We expect the connection string to be there.  If it doesn’t this will throw an error.  
    connectionString = connectionString.Substring(connectionString.IndexOf("=") + 1).Replace(@"""", string.Empty); 
```

DbUp 使用流畅的 API。我们需要告诉它我们的文件夹，每个文件夹的脚本类型，以及我们希望运行脚本的顺序。如果您使用带有 *StartsWith* 搜索的嵌入在汇编选项中的脚本，您需要在您的搜索中提供完整的名称空间。

```
var upgradeEngineBuilder = DeployChanges.To
    .SqlDatabase(connectionString, null)
    // Pre-deployment scripts, set them to always run first
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), x => x.StartsWith("DbUpSample.BeforeDeploymentScripts."), new SqlScriptOptions { ScriptType = ScriptType.RunAlways, RunGroupOrder = 0 })
    // Main Deployment scripts, they run once and run in the second group
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), x => x.StartsWith("DbUpSample.DeploymentScripts"), new SqlScriptOptions { ScriptType = ScriptType.RunOnce, RunGroupOrder = 1 })
    // Post deployment scripts, always run these scripts and run after everything has been deployed
    .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), x => x.StartsWith("DbUpSample.PostDeploymentScripts."), new SqlScriptOptions { ScriptType = ScriptType.RunAlways, RunGroupOrder = 2 })
    // By default all the scripts are run in the same transaction
    .WithTransactionPerScript()
    // Set this so it can report back to Octopus Deploy how things are going
    .LogToConsole();

var upgrader = upgradeEngineBuilder.Build();

Console.WriteLine("Is upgrade required: " + upgrader.IsUpgradeRequired()); 
```

升级程序已经构建好了，可以运行了。这一部分是我们注入升级报告参数检查的地方。如果设置了该参数，请不要运行升级。相反，为 Octopus Deploy 生成一个报告作为工件上传:

```
if (args.Any(a => a.StartsWith("--PreviewReportPath", StringComparison.InvariantCultureIgnoreCase)))
{
    // Generate a preview file so Octopus Deploy can generate an artifact for approvals
    var report = args.FirstOrDefault(x => x.StartsWith("--PreviewReportPath", StringComparison.OrdinalIgnoreCase));
    report = report.Substring(report.IndexOf("=") + 1).Replace(@"""", string.Empty);

    var fullReportPath = Path.Combine(report, "UpgradeReport.html");

    Console.WriteLine($"Generating the report at {fullReportPath}");

    upgrader.GenerateUpgradeHtmlReport(fullReportPath);
}
else
{
    var result = upgrader.PerformUpgrade();

    // Display the result
    if (result.Successful)
    {
        Console.ForegroundColor = ConsoleColor.Green;
        Console.WriteLine("Success!");
    }
    else
    {
        Console.ForegroundColor = ConsoleColor.Red;
        Console.WriteLine(result.Error);
        Console.WriteLine("Failed!");
    }
} 
```

当我们把它们放在一起时，它看起来像这样:

```
using System;
using System.IO;
using System.Linq;
using System.Reflection;
using DbUp;
using DbUp.Engine;
using DbUp.Helpers;
using DbUp.Support;

namespace DbUpSample
{
    class Program
    {
        static void Main(string[] args)
        {
            var connectionString = args.FirstOrDefault(x => x.StartsWith("--ConnectionString", StringComparison.OrdinalIgnoreCase));

            connectionString = connectionString.Substring(connectionString.IndexOf("=") + 1).Replace(@"""", string.Empty);

            var upgradeEngineBuilder = DeployChanges.To
                .SqlDatabase(connectionString, null)
                .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), x => x.StartsWith("DbUpSample.BeforeDeploymentScripts."), new SqlScriptOptions { ScriptType = ScriptType.RunAlways, RunGroupOrder = 0 })
                .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), x => x.StartsWith("DbUpSample.DeploymentScripts"), new SqlScriptOptions { ScriptType = ScriptType.RunOnce, RunGroupOrder = 1 })
                .WithScriptsEmbeddedInAssembly(Assembly.GetExecutingAssembly(), x => x.StartsWith("DbUpSample.PostDeploymentScripts."), new SqlScriptOptions { ScriptType = ScriptType.RunAlways, RunGroupOrder = 2 })
                .WithTransactionPerScript()
                .LogToConsole();

            var upgrader = upgradeEngineBuilder.Build();

            Console.WriteLine("Is upgrade required: " + upgrader.IsUpgradeRequired());

            if (args.Any(a => a.StartsWith("--PreviewReportPath", StringComparison.InvariantCultureIgnoreCase)))
            {
                // Generate a preview file so Octopus Deploy can generate an artifact for approvals
                var report = args.FirstOrDefault(x => x.StartsWith("--PreviewReportPath", StringComparison.OrdinalIgnoreCase));
                report = report.Substring(report.IndexOf("=") + 1).Replace(@"""", string.Empty);

                var fullReportPath = Path.Combine(report, "UpgradeReport.html");

                Console.WriteLine($"Generating the report at {fullReportPath}");

                upgrader.GenerateUpgradeHtmlReport(fullReportPath);
            }
            else
            {
                var result = upgrader.PerformUpgrade();

                // Display the result
                if (result.Successful)
                {
                    Console.ForegroundColor = ConsoleColor.Green;
                    Console.WriteLine("Success!");
                }
                else
                {
                    Console.ForegroundColor = ConsoleColor.Red;
                    Console.WriteLine(result.Error);
                    Console.WriteLine("Failed!");
                }
            }
        }
    }
} 
```

在启用报告参数的情况下运行控制台应用程序会生成我们预期的报告:

[![](img/17a106a95b5615ffd1b2081649ec2815.png)](#)

### 未来的工作

创建脚手架和编写 program.cs 文件的代码应该只需要做一次。有了我们的设置，你需要做的就是将文件添加到`PreDeployment`、`PostDeployment`和`Deployment`文件夹中。

使用这种设置，很容易删除旧文件，但是 DbUp 不喜欢这样做。DbUp 背后的想法是，它提供所有数据库更改的历史。例如，当您想要在新的开发人员的机器上创建一个新的数据库时，您只需要运行这个命令行应用程序。它将遍历并运行所有脚本，以启动并运行数据库。删除文件最终可能会删除一个键序列，例如创建一个表、添加一个关键列或者将列从一个表移动到另一个表。有这些额外的文件并不会对性能造成太大的影响。DbUp 将看到他们已经运行，并将他们从运行列表中排除。

您可以将较旧的文件移动到一个新的文件夹中，并添加一个新的命令行参数来选择这些文件。

## Octopus 部署配置

我将假设您知道如何构建一个. NET 核心应用程序并打包它。如果你没有，这里是快速 TL；博士:

*   在项目上运行`dotnet publish`命令(不要忘记输出路径)。
*   运行`octo pack`来打包输出路径(或者使用 Octopus Deploy build server 插件)。
*   使用`octo push`命令将包推送到 Octopus Deploy(或者使用 Octopus Deploy 构建服务器插件)。

为了让您的生活更轻松，对于这个演示，我在 GitHub repo 的根目录中以 zip 文件的形式包含了示例应用程序的 1.0.0.1 版本。将软件包上传到 Octopus 内置存储库:

[![](img/581ce2d92eba73af0375805199677ed1.png)](#)

我赞同 Octopus Deploy 项目应该负责自我引导的理论。对于数据库部署，这意味着在部署之前确保数据库存在，并创建必要的 SQL Server 用户。

在进入流程之前，我们需要定义一些变量。因为这是一篇博文的演示，所以我在所有环境中使用相同的数据库服务器:

[![](img/97303f9d03b15c8c62f04771b14f96ae.png)](#)

为了创建数据库和用户，这个过程将使用我为以前的博客文章创建的社区步骤模板。请参见[文档](https://octopus.com/docs/deployment-process/steps/community-step-templates#adding-community-step-templates)了解如何在你的 Octopus 服务器上下载和安装这些社区步骤模板。

我在**数据库工作者池**中的一个工作者上运行第一步。我选择使用单个工作池，因为我使用 SQL 身份验证。我不必担心每个环境的集成安全性和独特的服务帐户。如果我在这个过程中使用集成安全性和每个环境的唯一服务帐户，还有一些额外的设置要做，但是我将在后面的文章中介绍。现在，我想尽可能简单地解释一下:

[![](img/6005b495715d0b1a961af22d3f243fd6.png)](#)

首先，我们要搭建好脚手架，创建数据库、用户，并分配用户数据库:

[![](img/ce0b3fefa14f466de9b8c391eb672b13.png)](#)

下一组步骤将从 DbUp 部署数据库更改。如果你还记得[我之前的 Redgate 文章](/blog/database-deployment-automation-using-redgate-sql-change-automation)，这是分四步完成的:

1.  下载软件包。
2.  运行 Redgate 创建数据库版本。
3.  DBA 批准部署。
4.  运行 Redgate 部署数据库版本。

我对这个过程有几个问题。也就是说，下载包步骤被设计成提取包并把它留在触手上。默认情况下，它将永远保留在那里，除非配置了[保留策略](https://octopus.com/docs/administration/retention-policies)。部署完成后，没有必要将提取的包放在触手上。SQL 脚本已经运行，现在它们正在占用空间。此外，该流程的步骤 2 和 4 引用了步骤 1。感觉有很多额外的工作。

如果您正在使用 Octopus Deploy 2018.8 或更高版本，好消息是我们现在可以引用来自**运行脚本**步骤的包。包将被下载和提取，在步骤完成后，它将删除提取的包。除了清理不需要的内容，在**运行脚本**步骤中使用包引用非常适合在 workers 上运行部署，假设每个步骤都是独立的。这也意味着不同的工人可以完成每一步的工作。

该流程部署部分的第一步是生成 HTML 报告，并将其作为工件上传到 Octopus Deploy。点击**添加**按钮，将包引用添加到步骤中:

[![](img/ba3a0e6debcc13bb2088b7a71acaa598.png)](#)

当模式窗口出现时，选择要提取的包:

[![](img/0a44742c1443911d6f1d9a1935726143.png)](#)

现在我们可以添加一个脚本来处理部署。跑步。NET 核心控制台应用程序与运行。NET Framework 控制台应用程序。

这完全取决于您在构建和发布应用程序时设置的开关。您可以创建一个自包含的控制台应用程序(。exe)以及所有必要的。dll，但这样做会增加包的大小。或者，您可以将其设置为仅创建一个. dll，并引用所有外部依赖项。在示例包中，我创建了一个自包含的包，但是我排除了。zip 文件中的。这样，您就不必担心运行恢复了:

```
# How you reference the extracted path
$packagePath = $OctopusParameters["Octopus.Action.Package[DbUpSample].ExtractedPath"]
$connectionString = $OctopusParameters["Project.Database.ConnectionString"]
$reportPath = $OctopusParameters["Project.HtmlReport.Location"]

$dllToRun = "$packagePath\DbUpSample.dll"
$generatedReport = "$reportPath\UpgradeReport.html"

if ((test-path $reportPath) -eq $false){
    New-Item $reportPath -ItemType "directory"
}

# How you run this .NET core app
dotnet $dllToRun --ConnectionString="$connectionString" --PreviewReportPath="$reportPath"

New-OctopusArtifact -Path "$generatedReport" 
```

完成后，整个步骤如下所示:

[![](img/7599532b9eda32e592fb116f26479016.png)](#)

人工干预没什么特别的。在本例中，我将其配置为仅在试运行和生产环境中运行，并让数据库管理员批准该部署:

[![](img/d722fa9d7802b2cfeb2e9dd37ac999cf.png)](#)

部署步骤类似于**生成增量报告**步骤，除了它将部署变更，而不用担心报告的生成。这一步的 PowerShell 是:

```
# How you reference the extracted path
$packagePath = $OctopusParameters["Octopus.Action.Package[DbUpSample].ExtractedPath"]
$connectionString = $OctopusParameters["Project.Database.ConnectionString"]

$dllToRun = "$packagePath\DbUpSample.dll"

# How you run this .NET core app
dotnet $dllToRun --ConnectionString="$connectionString" 
```

[![](img/a8ab196dd09446898e8457d58fded9c0.png)](#)

现在最后的过程是:

[![](img/438d86955be37dcbbe869e09785e91ca.png)](#)

变数就在那里。流程设置完毕。让我们部署一些数据库更改:

[![](img/582cfd4d9495af4ecdd87260e44a9a4d.png)](#)

哎呦！忘记安装了。我的员工的网络核心:

[![](img/fa5715f6f8696bf835124e895e41c569.png)](#)

快速跳转到[脚本控制台](https://octopus.com/docs/administration/managing-infrastructure/script-console)来运行 chocolatey 安装:

[![](img/4fc484a948475868be7b96b277ace5bc.png)](#)

这是成功的:

[![](img/299592fc80d57966b3230b92e4808f13.png)](#)

让我们再试试那个版本。事后看来，我本可以告诉它重试发布，但我决定创建一个新的:

[![](img/2e70330e414753c2e24b174abce35cbd.png)](#)

这一次很成功。您可以看到由流程创建的工件。这是数据库管理员将下载并在试运行和生产中审查的内容:

[![](img/620ea976641a74770886b7ae49df3fbf.png)](#)

如果我们看一下数据库，我们会看到项目是按预期创建的:

【T2 ![](img/5e201b0f0ae6a7321945393b053edb5a.png)

## 综合安全和工人

在这个演示中，我使用了 SQL 身份验证。然而，你们中的许多人正在使用集成安全性。为了增加一层安全性，每个环境都有自己的 Active Directory 服务帐户。这完全有道理，我推荐这种方法。

对于现在这一代员工，你如何做到这一点？这并不像它应该的那样直截了当(我们希望在 workers v2 中解决这个问题)。我将指导您完成设置它的必要步骤。

首先，我们需要为每个环境创建一个专用的工作人员池。

[![](img/8c8ae6f574850cd87850e76678321e19.png)](#)

接下来，我们需要创建云区域部署目标。

您需要为每个环境创建一个云区域。我为这些云区域创建了一个名为`DbWorker`的新角色，因为我想要一种区分这些新部署目标的方法:

[![](img/e0416c8a9dadf9554f86faa9b56bdbc7.png)](#)

完成后，我有了四个新的云区域:

[![](img/4bef5ce31cd77c98ba4e547d34be378e.png)](#)

我将更改流程的执行位置，使其在该环境中使用`DbWorker`角色的目标上运行:

[![](img/3baf557b41d239fbf51a6125b9535010.png)](#)

对流程中每个步骤重复相同的更改:

[![](img/143afaa50e05e538a8395a900de1f9b4.png)](#)

当部署一个新版本来测试时，选择`Test Database Worker Region`:

[![](img/6804901eeed2b68a1f6455f978ec3126.png)](#)

## 结论

最近对 DbUp 的修改有助于为数据库创建一个健壮的部署管道。现在 DBA(和其他人)可以在部署之前通过 Octopus Deploy 检查变更。拥有审查变更的能力应该有助于在过程中建立信任，并有助于加速采用。

* * *

数据库部署自动化系列文章: