# 使用 Docker 作为包管理器- Octopus Deploy

> 原文：<https://octopus.com/blog/docker-as-package-manager>

自动化开发运维任务时的一个持续挑战是确保您拥有正确的工作工具。Linux 用户长期以来一直享受着从大量维护良好的软件包库中安装工具的能力，而 macOS 用户有 HomeBrew 和 MacPorts，Windows 用户有 Chocolatey 和 winget。

然而，越来越多的基于云的工具，例如特定于平台的 CLI 工具(例如 eksctl、aws-iam-authenticator、kubectl、kind 和 helm)，只能通过直接二进制下载跨多个操作系统和 Linux 发行版可靠地安装。在自动化开发运维任务时，寻找和下载工具是一个难点。

Docker 通过允许从 Docker 映像运行基于 CLI 的工具来提供解决方案。

在这篇文章中，我将探讨如何使用 Docker 作为一个通用的包管理器来跨多个操作系统下载和运行许多 CLI 工具。

DevOps 完全是关于可重复性和自动化，并且经常需要使用专门的 CLI 工具编写脚本任务。许多运行脚本的环境本质上都是短暂的。示例包括根据需要创建和销毁的 CI/CD 代理、用于运行作业的 Kubernetes pods，以及根据需求扩展和缩减的虚拟机。这通常意味着脚本编写者不能假定安装了 CLI 工具，或者依赖于工具的特定版本。

正如简介中提到的，DevOps 团队需要的许多工具只提供直接的二进制下载。这意味着即使是一个简单的脚本也可能需要首先找到任何所需 CLI 工具的相应下载链接，下载它们(通常使用某种重试逻辑来处理网络不稳定性)，提取或安装工具，然后最终运行它们。

由于从互联网下载文件所需的工具因操作系统而异，这一任务变得复杂。Linux 和 macOS 用户可能可以指望安装像`curl`或`wget`这样的工具，而 Windows 用户可能会使用 PowerShell CmdLets。

但是如果有更好的方法呢？

## Docker 作为一个通用的包管理器

如今，每个主要的 CLI 工具和平台都提供了维护良好的 Docker 映像。无论这些图片是由供应商自己发布在 Docker Hub 这样的存储库上，还是由第三方维护，比如 T2 Bitnami T3，你都很有可能找到你需要的最新版本的 CLI 工具作为 Docker 图片。

运行 Docker 镜像的好处在于，下载和执行镜像的命令在所有操作系统和所有工具上都是相同的。

要下载映像，重用任何以前下载的映像并自动重试，请运行命令:

```
docker pull image-name 
```

然后，使用以下命令执行该映像:

```
docker run image-name 
```

正如您将在接下来的部分中看到的，对于短期管理任务，可靠地运行映像还需要额外的参数，但是一般来说，对于每个工具和操作系统，您只需要知道这两个命令。与在 Windows 和 Linux 之间编写独特的脚本来从自定义 URL 下载二进制文件相比，这是一个巨大的进步。

让我们看看 helm，它是一个从 Docker 映像运行 CLI 工具的实际例子。第一步是在本地下载映像。注意这一步是可选的，因为 Docker 会在运行之前下载丢失的图像:

```
docker pull alpine/helm 
```

使用以下命令不带任何参数运行 helm。传递给`docker`的`--rm`参数在完成时清理容器，这在为单个操作运行图像时是理想的:

```
docker run --rm alpine/helm 
```

这导致帮助文本被打印到控制台，就像您运行了一个本地安装的没有参数的`helm`版本一样。

用于 DevOps 自动化的 CLI 工具的一个常见要求是能够读取文件和目录，无论它们是配置文件、较大的包(如 zip 文件)还是包含应用程序代码的目录。

就其本质而言，Docker 容器是自包含的，默认情况下不读取主机上的文件。然而，可以使用`-v`参数将本地文件和目录挂载到 Docker 容器中，允许进程在 Docker 容器中运行，以读写主机上的文件。

下面的命令用一个为运行`alpine/helm`映像而创建的容器挂载几个共享目录。这允许 Docker 容器中的`helm`可执行文件访问配置设置，比如 helm 存储库。它还传递参数`repo list`，其中列出了已配置的存储库:

```
docker run --rm -v "$(pwd):/apps" -w /apps \
    -v ~/.kube:/root/.kube -v ~/.helm:/root/.helm -v ~/.config/helm:/root/.config/helm \
    -v ~/.cache/helm:/root/.cache/helm \
    alpine/helm repo list 
```

如果您之前没有定义任何 helm 存储库，此命令的输出将是:

```
Error: no repositories to show 
```

为了演示卷挂载是如何工作的，使用本地安装版本的`helm`配置一个新的 helm 存储库:

```
helm repo add nginx-stable https://helm.nginx.com/stable 
```

运行 helm Docker 映像来列出存储库，这表明它已经加载了由本地安装的`helm`副本添加的存储库:

```
NAME            URL
nginx-stable    https://helm.nginx.com/stable 
```

反之亦然。运行以下命令，从 helm Docker 映像添加第二个存储库:

```
docker run --rm -v "$(pwd):/apps" -w /apps \
    -v ~/.kube:/root/.kube -v ~/.helm:/root/.helm -v ~/.config/helm:/root/.config/helm \
    -v ~/.cache/helm:/root/.cache/helm \
    alpine/helm repo add kong https://charts.konghq.com 
```

然后使用以下命令列出本地安装的工具中的 repos:

```
helm repo list 
```

本地安装反映了新添加的 repo:

```
NAME            URL
nginx-stable    https://helm.nginx.com/stable
kong            https://charts.konghq.com 
```

## 别名 Docker 运行命令

虽然 Docker 让您可以方便地下载和运行图像，但键入`docker run`命令可能会变得非常冗长乏味。幸运的是，可以给`docker run`命令起别名，这样它就可以替代本地安装的工具。

使用`alias`命令将`docker run`映射到`helm`命令:

```
alias helm='docker run --rm -v $(pwd):/apps -w /apps -v ~/.kube:/root/.kube -v ~/.helm:/root/.helm -v ~/.config/helm:/root/.config/helm -v ~/.cache/helm:/root/.cache/helm alpine/helm' 
```

先前的 alias 命令仅在定义它的会话中有效。如果您注销并重新登录，别名会丢失。要使别名永久化，编辑`~/.bash_aliases`文件，并在新的一行中添加上面的 alias 命令:

```
vim ~/.bash_aliases 
```

在许多 Linux 发行版中，`~/.bash_aliases`文件是由`~/.bashrc`文件自动加载的。如果没有，将以下代码添加到`~/.bashrc`文件中:

```
if [ -f ~/.bash_aliases ]; then
    . ~/.bash_aliases
fi 
```

## 在 Octopus 脚本步骤中使用 Docker 图像

如果您在非交互式 shell 中使用别名(比如在 Octopus 部署中运行脚本步骤)，您需要通过运行以下命令来确保别名得到扩展:

```
shopt -s expand_aliases 
```

以下代码片段显示了下载 Docker 映像并设置别名的脚本步骤。日志记录级别已设置为 verbose，因此 Docker 映像下载消息不会填充部署日志。它还演示了通过`-e`参数将环境变量传递给`docker run`:

```
echo "Downloading Docker images"

echo "##octopus[stdout-verbose]"

docker pull amazon/aws-cli 2>&1

# Alias the docker run commands
shopt -s expand_aliases
alias aws='docker run --rm -i -v $(pwd):/build -e AWS_DEFAULT_REGION=$AWS_DEFAULT_REGION -e AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID -e AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY amazon/aws-cli'

echo "##octopus[stdout-default]" 
```

## 结论

Docker 镜像为安装和运行通用 DevOps CLI 工具提供了一致且方便的方法，尤其是与编写 OS 命令从唯一的 URL 下载二进制文件相比。

在这篇文章中，我研究了如何从命令行或在脚本中运行 Docker 映像，并提供了一些技巧来允许 Docker 映像作为本地安装工具的替代运行。

## 了解更多信息

如果你想在 AWS 平台上构建和部署容器化的应用程序，比如 EKS 和 ECS，试试 Octopus Workflow Builder。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，展示漏洞扫描和基础架构即代码(IaC)等最佳实践。

愉快的部署！