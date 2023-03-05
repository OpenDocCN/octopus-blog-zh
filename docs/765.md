# RFC: Azure 和 FTP 步骤- Octopus 部署

> 原文：<https://octopus.com/blog/upcoming-features-azure-and-ftp>

我目前正在为 Octopus Deploy 构建两个新的步骤类型:

1.  **通过 FTP**
    部署在这一步中，我们将在 Octopus 服务器上提取一个 NuGet 包，然后将其“同步”到一个 FTP/FTPS 服务器上。这也使得部署 Azure 网站成为可能，因为它们支持 FTP。
2.  **部署 Azure 云服务包**
    在这一步中，我们将在 Octopus 服务上提取一个 NuGet 包，它应该包含一个 Azure 云服务包(。cspkg)(我们将扩展 Octo.exe 或 OctoPack，以便更容易地将. cspkg 重新打包为. nupkg)。然后，我们将 cspkg 上传到 blob 存储，然后将其部署到 Azure。

通常，我们在触手代理上运行包步骤。然而，在这种情况下，这两种步骤类型都将直接在 Octopus 服务器上运行，而不是在 Tentacles 上运行。这意味着如果你只是使用 Octopus 通过 FTP 或 Azure 进行部署，你不需要设置任何触手代理。

这两个步骤都将支持我们目前所做的常规的 XML 更新和转换，加上执行常规的 PowerShell 脚本(将在 Octopus 服务器上运行)。因此，举例来说，你可以在 FTP 之前或上传到 Azure 之前使用 PowerShell 更新文件。

配置 FTP 步骤时，您将设置主机、用户名、密码和其他常用设置。您也可以指定 FTP 根目录。所有这些都可以引用 Octopus 变量，例如，您可以为您的暂存/生产部署使用不同的主机/根目录/用户名。

对于 Azure 云服务包，我们还将提供两个选项来管理实例数量:

*   使用 CSCFG 中的实例计数设置
*   更新时保留当前部署的服务中的实例计数设置

例如，如果您最初使用两个实例部署到生产环境，然后使用 Azure 管理门户将其扩展到 4，我们可以使用 2(在 cscfg 中指定)或 4(我们将从管理 API 获取)，这取决于您选择的选项。

我们还将支持 Azure cscfg 文件中的变量替换。例如:

```
<Role name="WorkerRole1">
  <Instances count="1" />
  <ConfigurationSettings>
    <Setting name="MyConnectionString" value="blah blah" />
  </ConfigurationSettings>
</Role> 
```

如果您有一个名为`WorkerRole1/MyConnectionString`的 Octopus 变量，我们将替换该设置的“value”属性。请注意，该设置以角色名称为前缀，以允许不同角色的不同设置。

在选择要使用的云服务配置文件时，我们将寻找(按此顺序):

*   服务配置。*环境*。cscfg
*   服务配置。Cloud.cscfg

由于连接 Azure 管理 API 需要 X509 证书，我们将自动使用 Octopus 服务器证书。要允许 Octopus 部署到您的 Azure 订阅，您只需从 Octopus 下载`.cer`文件并将其上传到 Azure 门户。我们将在页面上提供到您的`.cer`和安装说明的链接，用于编辑 Azure 步骤。

你怎么想呢?正如我所说的，我目前正在构建这些功能，所以如果有其他与 Azure 或 FTP 相关的场景，我很乐意听到它们，这样我们就可以在下一个版本中包含它们。