# 将 Octopus 服务器移植到。网络核心 3.1 -八达通部署

> 原文：<https://octopus.com/blog/octopus-server-dotnet-core-lessons-learned>

[![Lessons learned porting Octopus Server to .NET Core 3.1](img/d474ecc0ec20eb01580b0656146e3e93.png)](#)

随着 Octopus 2020.1 的发布，Octopus Server 现在运行于。NET Core 3.1，这意味着它可以安装在 Linux、Docker 容器和 Kubernetes 上。这是一项重大的努力，我们已经分享了我们对推出章鱼云 1.0 的[思考，以及](/blog/octopus-cloud-1.0-reflections)[我们为什么选择 Kubernetes、Linux 和。NET Core for Octopus Cloud 2.0](/blog/octopus-cloud-v2-why-kubernetes) ，在本帖中，我们分享了这一变化的好处以及我们学到的三大经验教训。

## 利益

在我们分享经验教训之前，先来看看将 Octopus Server 移植到。网络核心:

*   现代开发环境、框架和工具。
*   对 Windows 和 Linux 的跨平台支持。
*   在 Kubernetes 中访问运行 Linux 容器的成熟生态系统。
*   选择和灵活性:我们的客户可以选择在 Windows 或 Linux 上运行 Octopus，也可以使用 Octopus Cloud。
*   Octopus Cloud 提高了性能，降低了运营成本。具体数字见上面的链接。

这也使得我们的开发环境更加高效，因为我们的工程团队现在可以选择在 Windows 或 Linux 上进行开发。

## 三大经验教训

通过这个过程，我们学到了很多东西；然而，我们学到了三大教训。

### 1.了解并计划 Windows 和 Linux 之间的差异

在的实现中存在特定于平台的差异。Windows 和 Linux 上的 NET Core。大多数问题都很小，有简单的解决方法，但我们确实发现了一些值得分享的重大问题。

**配置设置和 Windows 注册表**

为了同时支持 Windows 和 Linux 平台，我们必须删除任何特定于 Windows 的代码。Octopus Server 一开始是 Windows 产品，它遵循平台约定，在 Windows 注册表中存储了一些配置设置，这对 Linux 来说是个问题。

**解决方案**:

我们将注册表中存储的所有内容都转移到文件系统或 Octopus 数据库中。这是一个简单的任务，但需要时间和测试才能做好。

**数据库性能问题**

我们遇到的最大问题是糟糕的数据库性能，这是由于在 Windows 和 Linux 上处理数据库查询的方式不同。Octopus 使用 Microsoft SQL Server 作为其数据存储，我们在。NET 核心 SQL 客户端库。如果我们将`MultipleActiveResultSets`设置为`True`，我们会得到异常和数据库超时。上面链接的 GitHub 问题分享了完整的细节和一个简单的代码样本来重现问题。

**解决方案**:

我们的短期解决方案是禁用`MultipleActiveResultSets`设置，并尽量少用。通常，我们打开两个到数据库的连接，一个启用这个设置，另一个禁用它。我们主要使用禁用的连接，仅在需要时使用启用的连接。

我们一直在与微软合作，以帮助提供信息来解决这个问题，我们希望在未来看到一个适当的修复。

**认证提供者**

我们还遇到了在每个平台上托管不同的 Octopus 服务器 web 主机的需求。我们在 Windows 上使用`HttpSys`,在 Linux 上使用 *Kestrel* ,这使得我们的认证很有挑战性。Octopus 需要支持多种身份验证方案，包括基于 cookies 的身份验证，以及用户登录和注销并同时启用多个身份验证提供者的能力。

我们遇到的核心问题是`HttpSys`支持集成认证(即 Windows 认证)，但它是主机中每个端点的二进制开/关设置。这是不灵活的，这是从我们的非。NET 核心代码库。用户可以自动登录，但永远无法注销。

注意:我们不在 Windows 上使用 Kestrel，因为它不支持虚拟目录，而且我们的客户与其他服务共享同一个端口。因此，为了确保我们保持向后兼容性，我们决定只对 Windows 使用`HttpSys`。

**解决方案**:

我们考虑了几个选项，但是在经历了这个[ASP.NET 核心问题](https://github.com/dotnet/aspnetcore/issues/5888)之后，我们决定遵循那里的建议，使用两台主机。一个标准 web 主机，另一个主机看起来/表现得像主 API 站点根目录下的虚拟目录，即`/integrate-challenge`，因此与 Octopus Server 早期版本中的位置一致。主机只有这一条路由，当用户尚未通过身份验证时，它使用 401 响应发起登录质询。

### 2.提高您的 Linux 和 Docker 调试技能

当我们穿过。NET 核心端口，我们还学习了如何编码、测试和调试 Windows、Windows Subsystem for Linux (WSL)、Linux 和 Docker 的问题。从历史上看，我们的团队都是在 Windows 上开发的，但这已经演变为个人在 Windows、Linux 和 macOS 上编码，因此，我们学到了几个教训:

**以非超级用户或非管理员身份运行**

我们发现在 Linux 上以非 root 或非 sudo 身份运行所有构建和测试 Octopus Server 要容易得多，以限制基于权限的错误和问题。也就是说，我们有时需要使用`sudo`以 root 用户身份运行命令，然后用`chown`修改在这些命令中创建的文件的所有权，比如`sudo chown -R $user:$user ~/Octopus/MyInstance`。这有点快和脏，但它做的工作。

我们计划在未来改变这一点，例如 octopus 作为一个组的成员运行，在安装期间我们将`/etc/octopus`的组所有者配置为该组。

**证书管理**

我们的端到端(E2E)测试套件运行了一系列针对 Octopus 服务器监听 HTTPS(即端口 443)的测试。这需要我们将一些自签名证书转换并导入到 Linux 机器上`/etc/`的本地证书存储中。为了解决这个问题，我们编写了以下脚本:

```
#!/bin/bash
echo "Setup certificates"
CERTS_PATH_DEST="/usr/local/share/ca-certificates"
if [ ! -d "$CERTS_PATH_DEST" ]
then
    echo "Creating $CERTS_PATH_DEST"
    mkdir ${CERTS_PATH_DEST}
fi
MY_PATH="`dirname \"$0\"`"
CERT_PFX_FILES=(${PWD}/${MY_PATH}/../Octopus.E2ETests/Universe/*.pfx)
for CERT_PFX in "${CERT_PFX_FILES[@]}"
do
    FILE_NAME=$(basename "$CERT_PFX")
    CERT_CRT="${CERTS_PATH_DEST}/${FILE_NAME%.*}.crt"
    if [ ! -e "${CERT_CRT}" ]
    then
        echo "Converting '${CERT_PFX}' to '${CERT_CRT}'"
        openssl pkcs12 -in "${CERT_PFX}" -clcerts -nokeys -out "${CERT_CRT}" -passin pass:password
    fi
done
update-ca-certificates 
```

**连接到 SQL Server 数据库**

Octopus 使用 Microsoft SQL Server 作为其数据库，团队一般通过 Windows 服务器上的集成 Windows 身份验证连接到它。这不再管用了。我们这里的解决方案是切换到基于用户名和密码的身份验证。

此外，我们发现在关闭数据库连接池的情况下，我们的端到端(E2E)测试套件运行得更快、更可靠。我们还没有找到问题的根源，但是这可能是一个与上面提到的数据库性能问题相关的特定于平台的问题。

**用 Visual Studio 代码调试 Linux 上的 Octopus 服务器**

我们的团队使用各种工具编写代码，包括:

目前最流行的是带有[远程开发扩展](https://aka.ms/vscode-remote/download/extension)的 [Visual Studio 代码](https://code.visualstudio.com/)。这个扩展仍然是在预览，但我们发现它工作得很好。

使用 Visual Studio 代码和远程开发扩展，我们可以运行应用程序、测试和调试它们，以及在 Linux(或 Docker 容器)中编辑代码。只需将 VS Code 指向包含 Octopus 服务器代码的文件夹，然后只需按 F5 即可。就这么简单！

### 3.使用独立的包进行简化

把章鱼移植到。NET Core 允许我们发布[自包含包](https://www.hanselman.com/blog/MakingATinyNETCore30EntirelySelfcontainedSingleExecutable.aspx)，这带来了多重好处。

*   **更少的依赖性**:发布一个独立的可执行文件意味着我们不再需要。NET Core 安装在八达通服务器上。其结果是降低了安装要求，使八达通更容易安装。这是一个巨大的胜利。
*   **改进的可支持性**:简而言之，更少的依赖使得 Octopus 更容易安装和支持。零部件少了，能不小心改动的东西也少了。面向 [Windows](https://hub.docker.com/r/octopusdeploy/octopusdeploy) 和 Linux(即将推出)的 Docker 容器映像消除了更多的依赖，因为更多的依赖被内置到容器中。
*   **现代软件和工具**:使用现代工具和框架使我们的团队能够继续创新，并为我们的客户快速发布具有有用特性的软件。

不幸的是，这也有一些权衡。NET Core 3.1 要求我们[放弃对旧操作系统的支持](/blog/raising-minimum-requirements-for-octopus-server)，包括 Windows Server 2008-2012 和一些 Linux 发行版。支持旧的服务器和浏览器消耗了我们的时间和注意力，使我们更难创新和推动 Octopus 生态系统向前发展。

## 结论

我们把章鱼服务器移植到了。NET Core 3.1，以便服务器可以在 Linux、Docker 容器和 Kubernetes 上运行。我们作出这一改变是为了降低成本和提高我们的八达通云 SaaS 产品的性能，它取得了巨大的成功。

这不是一次简单的旅行，但我们在途中学到了很多东西。

1.  了解并计划 Windows 和 Linux 之间的差异
2.  提高您的 Linux 和 Docker 调试技能
3.  使用独立的包进行简化

## 相关职位