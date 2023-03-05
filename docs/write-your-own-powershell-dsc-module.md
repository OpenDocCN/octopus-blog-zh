# 编写自己的 PowerShell 期望状态配置(DSC)模块- Octopus Deploy

> 原文：<https://octopus.com/blog/write-your-own-powershell-dsc-module>

[![Octopus learning how to write a custom PowerShell DSC Module](img/53413df509ce497b29ac8d250c0d41db.png)](#)

PowerShell DSC 是一项非常棒的技术，可以放在您管理基于 Windows 的服务器的工具箱中。这篇文章是一系列文章的一部分:

我们也有关于使用 PowerShell DSC 和 Octopus Deploy 的文章:

* * *

随着您对 PowerShell 期望状态配置(DSC)的了解越来越多，您可能会遇到可用模块不太适合您想要做的事情的情况。您可以编写自己的[脚本资源](https://docs.microsoft.com/en-us/powershell/dsc/reference/resources/windows/scriptresource)，但是它们的伸缩性不好，传递参数很困难，并且它们不提供加密方法，以明文形式留下密码，但是，您可以编写自己的 DSC 模块。

在这个 PowerShell DSC 教程中，我将介绍如何编写您的第一个 PowerShell DSC 模块。

编写自己的 PowerShell DSC 模块并没有那么难。最困难的部分是将文件和文件夹放在正确的位置，因为 DSC 非常明确地规定了哪些文件放在哪里。然而，微软认识到这可能非常令人沮丧，并开发了 [xDscResourceDesigner](https://docs.microsoft.com/en-us/powershell/dsc/resources/authoringresourcemofdesigner) ，这是一个 PowerShell 模块，可以帮助您开始。使用这个模块，您可以很容易地定义您的资源需要什么属性，它将为您生成整个模块结构，包括 MOF 模式文件。如果你是第一次，我强烈推荐你使用这个模块，它可以帮你省去不少麻烦(相信我)。

## 安装 xDscResourceDesigner

安装模块与在系统上安装任何其他模块没有什么不同:

```
Install-Module -Name xDscResourceDesigner 
```

正如这篇[微软文章](https://docs.microsoft.com/en-us/powershell/dsc/resources/authoringresourcemofdesigner)所指出的，如果您的 PowerShell 版本早于第 5 版，您可能需要安装 PowerShellGet 模块以便安装工作。

## 使用 xDscResourceDesigner

使用 xDscResourceDesigner 实际上非常简单，只有两个函数:`New-DscResourceProperty`和`New-xDscResource`。`New-DscResourceProperty`是你用来定义你的 DSC 资源的属性。完成之后，将信息发送给`New-xDscResource`函数，它会生成实现资源所需的一切:

```
# Import the module for use
Import-Module -Name xDscResourceDesigner

# Define properties
$property1 = New-xDscResourceProperty -Name Property1 -Type String -Attribute Key
$property2 = New-xDscResourceProperty -Name Property2 -Type PSCredential -Attribute Write
$property3 = New-xDscResourceProperty -Name Property3 -Type String -Attribute Required -ValidateSet "Present", "Absent"

# Create my DSC Resource
New-xDscResource -Name DemoResource1 -Property $property1, $property2, $property3 -Path 'c:\Program Files\WindowsPowerShell\Modules' -ModuleName DemoModule 
```

这就是你自己的 DSC 模块，生成了所有的存根。

## 了解资源属性特性

对于 DSC 中资源属性的属性组件，有四个可能的值:

### 钥匙

使用您的资源的每个节点都必须有一个使该节点唯一的键。与数据库表类似，这个键不必是单个属性，但可以由几个属性组成，每个属性都带有 key 属性。在上面的例子中，`Property1`是我们的资源键。然而，也可以这样做:

```
# Define properties
$property1 = New-xDscResourceProperty -Name Property1 -Type String -Attribute Key
$property2 = New-xDscResourceProperty -Name Property2 -Type PSCredential -Attribute Write
$property3 = New-xDscResourceProperty -Name Property3 -Type String -Attribute Required -ValidateSet "Present", "Absent"
$property4 = New-xDscResourceProperty -Name Property4 -Type String -Attribute Key
$property5 = New-xDscResourceProperty -Name Property5 -Type String -Attribute Key 
```

在本例中，`property1`、`property4`和`property5`构成了节点的唯一值。键属性总是可写的，并且是必需的。

### 阅读

读取属性是只读的，不能为其赋值。

### 需要

必需属性是在声明配置时必须指定的可分配属性。使用上面的例子，当我们创建资源时，`Property3`属性被设置为 required。

### 写

写属性是可选属性，您可以在定义节点时为其指定值。在示例中，`Property2`被定义为写属性。

### ValidateSet 开关

`ValidateSet`开关可以与 Key 或 Write 属性一起使用，指定给定属性的允许值。在我们的例子中，我们已经指定`Property3`只能是`Absent`或`Present`。任何其他值都会导致错误。

## DSC 模块文件和文件夹结构

无论您决定[自己动手](https://docs.microsoft.com/en-us/powershell/dsc/resources/authoringResourceMOF)还是使用工具，文件夹和文件结构将如下所示:

```
$env:ProgramFiles\WindowsPowerShell\Modules (folder)
    |- DemoModule (folder)
        |- DSCResources (folder)
            |- DemoResource1 (folder)
                |- DemoResource1.psd1 (file, optional)
                |- DemoResource1.psm1 (file, required)
                |- DemoResource1.schema.mof (file, required) 
```

## MOF 文件

MOF 代表托管对象格式，是用于描述公共信息模型(CIM)类的语言。使用来自*工具的示例来帮助您编写模块*部分，生成的 MOF 文件将如下所示:

```
[ClassVersion("1.0.0.0"), FriendlyName("DemoResource1")]
class DemoResource1 : OMI_BaseResource
{
    [Key] String Property1;
    [Write, EmbeddedInstance("MSFT_Credential")] String Property2;
    [Required, ValueMap{"Present","Absent"}, Values{"Present","Absent"}] String Property3;
}; 
```

MOF 文件将只包含我们将在模块中使用的属性，以及它们的属性和数据类型。除非我们添加或删除属性，否则这几乎是我们对 MOF 文件所做的全部工作。

## psm1 文件

psm1 文件是我们大部分代码将要存放的地方。该文件将包含三个必需的函数:

*   `Get-TargetResource`
*   `Test-TargetResource`
*   `Set-TargetResource`

### 获取目标资源

`Get-TargetResource`函数返回资源所负责的当前值。我们使用`xDscResourceDesigner`得到的存根函数如下所示:

```
function Get-TargetResource
{
    [CmdletBinding()]
    [OutputType([System.Collections.Hashtable])]
    param
    (
        [parameter(Mandatory = $true)]
        [System.String]
        $Property1,

        [parameter(Mandatory = $true)]
        [ValidateSet("Present","Absent")]
        [System.String]
        $Property3
    )

    #Write-Verbose "Use this cmdlet to deliver information about command processing."

    #Write-Debug "Use this cmdlet to write debug information while troubleshooting."

    <#
    $returnValue = @{
    Property1 = [System.String]
    Property2 = [System.Management.Automation.PSCredential]
    Property3 = [System.String]
    }

    $returnValue
    #>
} 
```

注意，该功能不需要可选参数`Property2`(写入属性)。

### 测试目标资源

`Test-TargetResource`函数返回一个布尔值，表明资源是否处于期望的状态。从我们生成的示例来看，该函数如下所示:

```
function Test-TargetResource
{
    [CmdletBinding()]
    [OutputType([System.Boolean])]
    param
    (
        [parameter(Mandatory = $true)]
        [System.String]
        $Property1,

        [System.Management.Automation.PSCredential]
        $Property2,

        [parameter(Mandatory = $true)]
        [ValidateSet("Present","Absent")]
        [System.String]
        $Property3
    )

    #Write-Verbose "Use this cmdlet to deliver information about command processing."

    #Write-Debug "Use this cmdlet to write debug information while troubleshooting."

    <#
    $result = [System.Boolean]

    $result
    #>
} 
```

### 设置-目标资源

`Set-TargetResource`功能用于将资源配置到指定的期望状态。我们生成的示例如下所示:

```
function Set-TargetResource
{
    [CmdletBinding()]
    param
    (
        [parameter(Mandatory = $true)]
        [System.String]
        $Property1,

        [System.Management.Automation.PSCredential]
        $Property2,

        [parameter(Mandatory = $true)]
        [ValidateSet("Present","Absent")]
        [System.String]
        $Property3
    )

    #Write-Verbose "Use this cmdlet to deliver information about command processing."

    #Write-Debug "Use this cmdlet to write debug information while troubleshooting."

    #Include this line if the resource requires a system reboot.
    #$global:DSCMachineStatus = 1
} 
```

## 摘要

无论简单还是复杂，创建自己的 PowerShell DSC 模块的步骤都是一样的。这篇文章旨在让你朝着正确的方向开始。从这里，您可以创建您的模块来适应您需要配置的任何资源，并保持在期望的状态。关于工作模块的完整示例，请查看我的 GitHub repo 上的 [xCertificatePermission](https://github.com/twerthi/xCertificatePermission) 。