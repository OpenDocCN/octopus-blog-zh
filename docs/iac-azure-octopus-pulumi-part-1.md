# 使用 Octopus Deploy 和 Pulumi 将基础设施作为代码:第一部分——Octopus Deploy

> 原文：<https://octopus.com/blog/iac-azure-octopus-pulumi-part-1>

[![Infrastructure as code in Azure with Octopus Deploy and Pulumi: Part one](img/5afc6725213591e187844b05a34cee96.png)](#)

Pulumi 是一个基础设施代码解决方案，允许你用你已经熟悉的语言定义你的基础设施，比如 Go、Python 或 JavaScript。

在这篇文章中，我将向您展示如何使用用 Go (Golang)编写的 Pulumi 项目创建 Azure 资源组，以及如何使用 Octopus Deploy 部署它。

## 先决条件

要跟进这篇文章，你需要:

*   一个 [GitHub 账号](https://www.github.com)。
*   至少一个 Linux 部署目标。
*   Azure 订阅。
*   在 Octopus 中配置的 Azure 帐户。
*   一个免费的账户。

## 为什么普鲁米和章鱼部署？

Pulumi 是一个多语言云开发平台，允许您使用编程语言(如 Go、C#、Python、TypeScript、JavaScript、F#、VB)来构建云服务。无论您是想构建虚拟机、网络、无服务器实现，还是其他任何东西，Pulumi 都可以提供帮助。

对于 Pulumi 支持的每种语言，都有一个 SDK 可以用来与不同的云服务进行交互。例如，您可以使用 Azure SDK 创建一个资源组。

使用 Octopus Deploy，您可以使用 community steps 在 Windows 和 Linux 服务器上运行 Pulumi 项目。这应该涵盖您需要工作的任何环境。

### 创建 Pulumi 项目

1.  登录到 Pulumi。

2.  点击 **+新建项目**。

3.  选择 **Azure** 为云。

4.  选择语言的**转到**，点击**下一步**。

5.  添加项目的详细信息。您可以保留默认值，也可以添加自定义元数据。

    *   **项目名称**:正在创建的项目的名称。
    *   **项目描述**:正在创建的项目的描述。
    *   **栈**:栈名(dev，prod 等。).
    *   **配置**:对于配置，您会看到几种不同的类型可供使用:
        *   公共
        *   美国政府
        *   德国的
        *   瓷器；（China）中国

    对于一个开发环境，只要你没有任何规定，保持它`public`就可以了。完成后，点击**创建项目**。

6.  如果您还没有在本地安装 Pulumi，您应该现在安装它。你可以在 Windows 上使用 [Chocolatey](https://chocolatey.org/) ，或者在 MacOs 上使用 [Homebrew](https://brew.sh/) 。

7.  为您的项目创建一个新目录，并将目录(`cd`)更改为新创建的项目目录。

8.  接下来，将项目从 Pulumi 拉到您刚刚创建的目录中，运行以下命令，然后按照说明进行操作:

`pulumi new azure-go -s AdminTurnedDevOps/azure-go-new-resource-group/dev`

9.  从 Pulumi 中取出项目后，就该部署它了:

`pulumi up`

## 使用 Pulumi 编写代码

现在您已经创建了 Pulumi 项目，并且它在您的本地机器上是可用的，您已经拥有了使用 Go 与 Pulumi 进行交互所需要的一切。

该项目包括:

*   一个 go.mod 文件，它指定了所需的包。
*   指定项目和项目名称的 YAML 配置文件。
*   main.go 文件，其中已经包含了 go 代码。

对于每个 Pulumi 项目，默认情况下您会看到 starter 代码，它会向您显示使用了哪些 SDK 和软件包。

### Azure 示例

让我们从头开始创建一些东西，而不是使用 main.go 文件中的默认代码。

首先要指定的是包名和导入。因为代码来自`main`，所以您使用的包也将是`main`。

从标准库中，`fmt`和`log`将用于打印输出到屏幕上。这两个 Pulumi 包分别用于 Azure SDK 和 Pulumi SDK:

```
package main

import (
    "fmt"
    "log"

    "github.com/pulumi/pulumi-azure/sdk/v3/go/azure/core"
    "github.com/pulumi/pulumi/sdk/v2/go/pulumi"
) 
```

接下来，创建包含三个参数的资源组函数:

*   来自 Pulumi 的上下文
*   资源组名称
*   位置

```
func newResourceGroup(ctx *pulumi.Context, resourceGroupName string, location string) {
    if ctx == nil {
        log.Println("Pulumi CTX is not working as expected... please check issues on the SDK: github.com/pulumi/pulumi/sdk/v2/go/pulumi")
    } else {
        pulumi.Run(func(ctx *pulumi.Context) error {
            resourceGroup, err := core.NewResourceGroup(ctx, resourceGroupName, &core.ResourceGroupArgs{Location: pulumi.String(location)})
            if err != nil {
                log.Println(err)
            }

            fmt.Println(resourceGroup)
            return nil
        })
    } 
```

`if`语句检查`ctx`是否是`nil`，并允许我们查看 SDK 在上下文中是否有问题。

`else`语句包括执行 Pulumi 程序主体的`Run()`函数。正如你在 [GitHub](https://github.com/pulumi/pulumi/blob/master/sdk/go/pulumi/run.go) 上的 SDK 中看到的，它需要一个匿名函数，特别是在上下文中传递。

代码的核心在`core.NewResourceGroup`中，它创建了资源组。您还可以添加一些错误处理:

```
func main() {
    resourceGroupName := "octopuspulumitest"
    location := "eastus"
    ctx := &pulumi.Context{}

    newResourceGroup(ctx, resourceGroupName, location)

} 
```

`main`函数执行`newResourceGroup()`函数，并在运行时传入一些参数。还有一个 Pulumi 上下文的空初始化，因为这是`newResourceGroup()`中需要的参数之一。

完成的代码片段如下所示:

```
package main

import (
    "fmt"
    "log"

    "github.com/pulumi/pulumi-azure/sdk/v3/go/azure/core"
    "github.com/pulumi/pulumi/sdk/v2/go/pulumi"
)

func main() {
    resourceGroupName := "octopuspulumitest"
    location := "eastus"
    ctx := &pulumi.Context{}

    newResourceGroup(ctx, resourceGroupName, location)

}

func newResourceGroup(ctx *pulumi.Context, resourceGroupName string, location string) {
    if ctx == nil {
        log.Println("Pulumi CTX is not working as expected... please check issues on the SDK: github.com/pulumi/pulumi/sdk/v2/go/pulumi")
    } else {
        pulumi.Run(func(ctx *pulumi.Context) error {
            resourceGroup, err := core.NewResourceGroup(ctx, resourceGroupName, &core.ResourceGroupArgs{Location: pulumi.String(location)})
            if err != nil {
                log.Println(err)
            }

            fmt.Println(resourceGroup)
            return nil
        })
    }
} 
```

## 结论

最初配置 Pulumi 有多个步骤，但正如您所见，它非常强大。您可以使用您喜欢使用的编程语言，并在一个环境中创建您需要的基础设施或服务。

在我的下一篇[帖子](/blog/iac-azure-octopus-pulumi-part-2)中，我将向您展示如何打包 Go 代码并使用 Octopus Deploy 部署它。

## 观看网络研讨会

[https://www.youtube.com/embed/SA3-efF5PWk](https://www.youtube.com/embed/SA3-efF5PWk)

VIDEO

我们定期举办网络研讨会。请参见第[页的](https://octopus.com/events)网上研讨会，了解以往网上研讨会的档案以及即将举办的网上研讨会的详细信息。

愉快的部署！