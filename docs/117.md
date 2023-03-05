# 使用 Octopus API - Octopus Deploy 克隆空间

> 原文：<https://octopus.com/blog/cloning-a-space-with-octopus-api>

[![Cloning a space using the Octopus API](img/b649d6a401e6c34cce140b898ecb4c4a.png)](#)

我很高兴[为我与 Redgate](https://samples.octopus.app/app#/Spaces-106/projects/redgate-feature-branch-example/deployments) 的[网络研讨会收集了分支示例](https://event.on24.com/eventRegistration/EventLobbyServlet?target=reg20.jsp&partnerref=OS&eventid=2307799&sessionid=1&key=DCB6FD8458D78BEEBB341BE31CF8279B&regTag=&sourcepage=register)。我在 Redgate 的合作伙伴非常喜欢它，他们想要一个自己的副本来四处看看。那个示例项目位于 Octopus Cloud 上，所以我不能使用[迁移器](https://octopus.com/docs/administration/data/data-migration)，老实说，我也不想为这个特定的用例这么做。对于那些不知道的人来说，Octopus Web 门户中所做的一切都涉及 Octopus Deploy RESTful API。我开始编写一个 PowerShell 脚本，使用 Octopus API 将一个项目从 sample 实例复制到 Redgate 的实例中。在这篇文章中，我分享了我是如何做到这一点的，以及你可以做些什么来编写自己的脚本。

**TL；DR；**我在章鱼样本组织下的 [GitHub 上发布了我的脚本。接下来，我们的团队将在内部使用这个脚本来加快在那个实例中创建样本的速度。你可以自己试一试，叉一叉，修改一下，满足你公司的需求。](https://github.com/OctopusSamples/SpaceCloner)

捆绑的 [Octopus 数据迁移](https://octopus.com/docs/administration/data/data-migration)工具完全按照它在 tin 上显示的那样做。它帮助将 Octopus 项目从一个实例迁移到另一个实例。但是我经常看到人们试图将工具用于非设计用途的用例。这是一种钝器。您给它一个项目名称，它就变成了一个巨大的真空吸尘器，吸取关于该项目的所有东西，并导出它，准备导入到另一个实例中。

对于我的用例，迁移器有几个限制:

*   它必须与 Octopus Deploy 实例运行在同一个服务器上，因为它需要直接访问数据库来检索敏感变量。它被设计成*只为迁移*工作，这意味着所有敏感的变量都会随之而来。然而，我无权访问云实例(它们运行在 Kubernetes 容器中)。
*   对目标实例上部署流程的任何更新都将被覆盖。想象一下，我在 Redgate 的朋友选择修改他们的部署过程来满足他们自己的特定需求。他们要求重新同步。迁移者会进来用核武器摧毁他们的任何改变。
*   它不保留部署过程中的现有步骤。如果我在 Redgate 的朋友选择更改工人池名称或变量名，一个新的同步也会覆盖它。
*   它为后续运行导出了太多数据。在初始运行之后，我不再关心生命周期阶段、工人池、环境或任何类似的东西。我只想确保他们拥有部署流程中的最新步骤以及任何缺失的变量。

有了 PowerShell、Octopus Deploy API 和一些 if/then 语句(好的，有很多 if/then 语句)，我可以实现 90-95%的目标。

## Octopus 部署 API 的局限性

Octopus Deploy API 无法解密敏感变量。这包括帐户变量、标记为敏感的变量以及外部源用户名和密码。对于我的用例，所有这些都是完全可以接受的。我不想把所有敏感信息都给雷德盖特。他们有自己的数据库服务器，自己的 AWS 云账号等等。

像任何 RESTful API 一样，它不太擅长以高效的方式处理大量的 BLOB 数据，比如包、任务日志、工件、项目图像和租户图像。最后，这对于我的用例来说很好，因为这些项目不是我想克隆到 Redgate 实例的东西。他们想要这个过程，但是他们不关心版本、部署、快照等等。

## 脚本的目标

如果您编写的代码试图满足所有人的所有需求，那么您最终得到的代码几乎不能为任何人工作，或者它是一个没有人能够维护或理解的复杂的混乱。

我剧本的目标是:

1.  支持多次运行-我应该能够运行脚本多次，而不是让它核武器和铺平一切视线。
2.  挑选要克隆的项目——一开始，我想克隆很多数据来打基础。之后，我只想克隆数据的一个子集。
3.  保持简单愚蠢(吻)——不要试图依赖*的树*走路。按名称查找项目，如果找到匹配项，很好，就使用它。假设数据不在那里，并处理它。
4.  关注这个用例——我不得不多次提醒自己只关注特定的用例。不要试图让它适用于所有场景。只做好一个场景。

## API 基础

当我写剧本的时候，我注意到了一些通用规则，我想分享一下以助一臂之力。

### 相同的属性集

所有核心对象(环境、项目、工人等。)具有相同的三个属性:

它们将被命名为。我编写了几个助手函数来帮助在列表中按 ID 或名称查找项目:

```
function Get-OctopusItemByName
{
    param (
        $ItemList,
        $ItemName
        )    

    return ($ItemList | Where-Object {$_.Name -eq $ItemName})
}

function Get-OctopusItemById
{
    param (
        $ItemList,
        $ItemId
        ) 

    Write-VerboseOutput "Attempting to find $ItemId in the item list of $($ItemList.Length) item(s)"

    foreach($item in $ItemList)
    {
        Write-VerboseOutput "Checking to see if $($item.Id) matches with $ItemId"
        if ($item.Id -eq $ItemId)
        {
            Write-VerboseOutput "The Ids match, return the item $($item.Name)"
            return $item
        }
    }

    Write-VerboseOutput "No match found returning null"
    return $null    
} 
```

### Links 对象

从 Octopus Deploy API 返回的所有对象都有一个 Links 对象。当您需要更新现有项目或需要转到列表中的下一页时，这非常有用。

[![](img/893fee7f7e6f4833162fdcdd3203e72b.png)](#)

### 一致的端点参数

对于我需要创建或查询的项目，我注意到 Octopus Deploy 的 API 遵循相同的规则集。

因此，我能够将一些辅助函数放在一起查询 Octopus API:

```
function Get-OctopusUrl
{
    param (
        $EndPoint,        
        $SpaceId,
        $OctopusUrl
    )  

    if ($EndPoint -match "/api")
    {        
        return "$OctopusUrl/$endPoint"
    }

    if ([string]::IsNullOrWhiteSpace($SpaceId))
    {
        return "$OctopusUrl/api/$EndPoint"
    }

    return "$OctopusUrl/api/$spaceId/$EndPoint"
}

function Invoke-OctopusApi
{
    param
    (
        $url,
        $apiKey,
        $method,
        $item
    )

    try 
    {
        Write-VerboseOutput "Invoking $method $url"

        if ($null -eq $item)
        {            
            return Invoke-RestMethod -Method $method -Uri $url -Headers @{"X-Octopus-ApiKey"="$ApiKey"}
        }

        $body = $item | ConvertTo-Json -Depth 10
        Write-VerboseOutput $body
        return Invoke-RestMethod -Method $method -Uri $url -Headers @{"X-Octopus-ApiKey"="$ApiKey"} -Body $body
    }
    catch 
    {
        $result = $_.Exception.Response.GetResponseStream()
        $reader = New-Object System.IO.StreamReader($result)
        $reader.BaseStream.Position = 0
        $reader.DiscardBufferedData()
        $responseBody = $reader.ReadToEnd();
        Write-VerboseOutput -Message "Error calling $url $($_.Exception.Message) StatusCode: $($_.Exception.Response.StatusCode.value__ ) StatusDescription: $($_.Exception.Response.StatusDescription) $responseBody"        
    }

    Throw "There was an error calling the Octopus API please check the log for more details"
}

Function Get-OctopusApiItemList
{
    param (
        $EndPoint,
        $ApiKey,
        $SpaceId,
        $OctopusUrl
    )    

    $url = Get-OctopusUrl -EndPoint $EndPoint -SpaceId $SpaceId -OctopusUrl $OctopusUrl    

    $results = Invoke-OctopusApi -Method "Get" -Url $url -apiKey $ApiKey

    Write-VerboseOutput "$url returned a list with $($results.Items.Length) item(s)" 

    return $results.Items
}

Function Get-OctopusApi
{
    param (
        $EndPoint,
        $ApiKey,
        $SpaceId,
        $OctopusUrl
    )    

    $url = Get-OctopusUrl -EndPoint $EndPoint -SpaceId $SpaceId -OctopusUrl $OctopusUrl    

    $results = Invoke-OctopusApi -Method "Get" -Url $url -apiKey $ApiKey

    return $results
}

Function Save-OctopusApi
{
    param (
        $EndPoint,
        $ApiKey,
        $Method,
        $Item,
        $SpaceId,
        $OctopusUrl
    )

    $url = Get-OctopusUrl -EndPoint $EndPoint -SpaceId $SpaceId -OctopusUrl $OctopusUrl 

    $results = Invoke-OctopusApi -Method $Method -Url $url -apiKey $ApiKey -item $item

    return $results
}

function Save-OctopusApiItem
{
    param(
        $Item,
        $Endpoint,
        $ApiKey,
        $SpaceId,
        $OctopusUrl
    )    

    $method = "POST"

    if ($null -ne $Item.Id)    
    {
        Write-VerboseOutput "Item has id, updating method call to PUT"
        $method = "Put"
        $endPoint = "$endPoint/$($Item.Id)"
    }

    $results = Save-OctopusApi -EndPoint $Endpoint $method $method -Item $Item -ApiKey $ApiKey -OctopusUrl $OctopusUrl -SpaceId $SpaceId

    Write-VerboseOutput $results

    return $results
} 
```

### 不要忘记伐木

起初，我没有太多的记录。这是一个错误，但我很快意识到我需要记录一切。不仅如此，记录发送到 API 的 JSON 请求。这使得调试进行得更快。

我想让用户很容易知道一般的信息性消息、警告和错误。我用简单的`Green`表示好，`Yellow`表示警告，`Red`表示坏。我还添加了一个详细日志，将所有内容写到一个文件中。我创建了几个有用的函数来帮助日志记录:

```
$currentDate = Get-Date
$currentDateFormatted = $currentDate.ToString("yyyy_MM_dd_HH_mm")
$logPath = "$PSScriptRoot\Log_$currentDateFormatted.txt"
$cleanupLogPath = "$PSScriptRoot\CleanUp_$currentDateFormatted.txt"

function Write-VerboseOutput
{
    param($message)

    Add-Content -Value $message -Path $logPath    
}

function Write-GreenOutput
{
    param($message)

    Write-Host $message -ForegroundColor Green
    Write-VerboseOutput $message    
}

function Write-YellowOutput
{
    param($message)

    Write-Host $message -ForegroundColor Yellow    
    Write-VerboseOutput $message
}

function Write-RedOutput
{
    param ($message)

    Write-Host $message -ForegroundColor Red
    Write-VerboseOutput $message
}

function Write-CleanUpOutput
{
    param($message)

    Write-YellowOutput $message
    Add-Content -Value $message -Path $cleanupLogPath
} 
```

### 翻译 ID 值

源实例的`Production`可能有`Environments-123`，而目的地被设置为`Environments-555`。要进行翻译，您需要:

1.  将源 ID 转换为名称。
2.  将名称转换为目的地 ID。

同样，我为此编写了一个助手函数:

```
function Convert-SourceIdToDestinationId
{
    param(
        $SourceList,
        $DestinationList,
        $IdValue
    )

    Write-VerboseOutput "Getting Name of $IdValue"
    $sourceItem = Get-OctopusItemById -ItemList $SourceList -ItemId $IdValue
    Write-VerboseOutput "The name of $IdValue is $($sourceItem.Name)"

    Write-VerboseOutput "Attempting to find $($sourceItem.Name) in Destination List"
    $destinationItem = Get-OctopusItemByName -ItemName $sourceItem.Name -ItemList $DestinationList
    Write-VerboseOutput "The destination id for $($sourceItem.Name) is $($destinationItem.Id)"

    if ($null -eq $destinationItem)
    {
        return $null
    }
    else
    {
        return $destinationItem.Id
    }
} 
```

### 走阻力最小的路

对于您的脚本，首先关注最重要的项目。完美是完成的敌人。

我在 API 脚本中遇到了一些棘手的问题。具体围绕:

*   脚本包引用
*   将手动干预分配给团队
*   租户变量
*   处理不同版本的 Octopus Deploy(从 2020.2 克隆到 2020.1)

我选择走阻力最小的路。我要么写代码删除项(包引用)，设置默认值(手动干预)，跳过它们(租户变量)，要么添加一个 guard 子句阻止它运行(不同版本的 Octopus Deploy)。如果我对这些特定的项目有足够高的需求，我可能会考虑添加它们。但是现在，让他们工作的复杂性与回报的对比是不值得的。

## 收集数据

我的目标是将该项目从我们的 samples 实例克隆到一个新实例上。一个项目不仅仅是一个*项目*。它依赖于 Octopus 中的许多其他数据，所以我将重点放在我在建立新空间或新实例时必须创建的项目上。

这是我得到的数据列表:

*   环境
*   工人池(不是工人，只是池)
*   项目组
*   外部源
*   租户标签
*   步骤模板(社区和自定义步骤模板)
*   帐目
*   库变量集
*   生活过程
*   项目
    *   设置
    *   部署流程
    *   运行手册
    *   变量
*   租户(无租户变量)

我故意在清单上遗漏了一些关键项目。有些可能会让你吃惊:

*   目标
*   工人
*   部署
*   放
*   包装
*   用户
*   组
*   角色
*   外部身份验证提供者

我的用例是将一个项目从我的 samples 实例复制到 Redgate 运行的一个新实例中。上面列表中的每一项在我们的两个实例中都是不同的。我想保持简单。我将它们标记为已排除，然后继续。

## 敏感值和虚拟数据

我知道我会碰到一些敏感的价值观。我也知道我得不到实际价值，也不想得到。然而，在很多情况下，特别是账户，他们希望输入*的东西*。我在脚本中创建了以下两条规则:

1.  如果数据不存在，输入虚拟数据。如果可能，使用`DUMMY VALUE`。
2.  如果数据确实存在，就不要管它。不要试图覆盖它。

## 过滤数据

我想克隆的每个项目基本上都遵循了这个工作流程。我将以环境为例:

1.  加载源中的所有环境。
2.  加载目标中的所有环境。
3.  使用用户提供的过滤器过滤源环境。
4.  将过滤后的列表与目的地进行比较，如果存在，则跳过，如果不存在，则创建。

对于一些物体来说，这要复杂得多。我必须转换某些属性，或者删除某些属性。在某些情况下，我想覆盖现有的数据。

保持过滤器简单，但功能强大，对我来说至关重要。我选择了以下方式:

*   `All` -关键字，将尝试克隆该对象的所有数据。
*   逗号分隔列表——指定一个 CSV，比如`test,staging,production`,它将克隆这三个特定的值。
*   使用正则表达式的通配符支持——可以与 CSV 结合使用，因此您可以为变量集指定`AWS*,Notification`,这将包含所有以 AWS 和通知变量集开头的变量集。

## 变量和部署流程

变量和部署过程是克隆中最复杂的部分。最初的运行是直接的，把源中的内容复制到目的地。

随后的运行是棘手的。我从三个基本规则开始:

1.  Redgate 很可能修复了最初由我的脚本创建的所有虚拟数据。
2.  Redgate 不愿意一遍又一遍地修复我的脚本创建的相同虚拟数据。
3.  Redgate 很可能会以某种方式改变变量和部署过程。

你可以用*我的用户*替换 *Redgate* 作为你自己的脚本。我想对你来说也是如此。

源项目是真理的来源。我写这个脚本是为了遵循这些规则:

1.  遍历源数据。对于每一项，检查数据是否存在于目标上，如果不存在，从源克隆。否则，使用现有数据。
2.  遍历目标数据。对于每一项，检查源代码，看它是否存在。如果源中不存在该项目，则将其添加回来。

我们用一个实实在在的例子。我在目标实例上有一个部署过程。我的目标实例上的部署流程有一个不在源中的新步骤。

为了清楚起见，目标实例使用黑暗模式。源实例正在使用光照模式。

[![](img/1e40345102fe635d8e5ada545f54aef5.png)](#)

我的源实例上的流程有步骤 3，它没有出现在目标流程中。

[![](img/61a279b7653d7bcc8548816e944531fa.png)](#)

同步运行后的过程遵循上述规则，现在看起来如下所示。来自源的新步骤 3 被添加到适当的位置，然后仅在目标上找到的新步骤被添加到后面。

[![](img/aaa3bf3a2e8de7d887af56859f9e7d9d.png)](#)

## 扩展使用案例

当我写完我的脚本时，我意识到我可以支持迁移器之外的许多用例。这些使用案例包括:

*   从云复制到自托管——我家里有一个管理程序，有一个我可以克隆到的地方将使本地测试变得更容易。
*   从自托管复制到云——在我完成修改后，从我的虚拟机管理程序复制到示例实例。
*   创建新空间时复制默认变量集——客户成功团队在我们的示例实例中创建了许多新空间。
*   将一个巨大的空间分割成几个小空间——我们已经在我们的示例实例中这样做了几次，有了它会使事情变得容易得多。
*   保持父/子项目过程同步——这是我们几个示例中常见的一个，我们将克隆一个项目，并有一个不同的目标(使用 AWS 而不是 Azure)。通过脚本保持两个项目同步将使我们的生活更加轻松。

## 帮助入门的示例

我意识到从头开始写剧本是一项艰巨的任务。这就是为什么我发布了我的团队 Customer Success 用来管理我们的示例实例的脚本。你可以在 Octopus Samples 组织中找到那个[项目。](https://github.com/OctopusSamples/SpaceCloner)

我希望你会叉回购，并修改它，以满足您的特定需求。也许您想要克隆目标，或者也许您想要排除所有变量。这不是我们计划对该脚本进行的更改，但是通过分叉，您可以对该脚本做任何您喜欢的事情。希望该脚本有足够的内容来帮助您满足基本的克隆需求。

下次再见，愉快的部署！