# 特性分支 web 应用程序- Octopus 部署

> 原文：<https://octopus.com/blog/feature-branch-web-apps>

[![octopus hanging off a branch in a pot plant](img/3a9386c8f6b779c51d8adbc5afaa74d8.png)](#)

Octopus 非常擅长管理开发、测试和生产环境中的变更进度。它还通过使用通道很好地处理了分支策略，如 hotfixes，允许您绕过某些环境，在紧急情况下将具有匹配版本规则的包(如在版本发布字段中有单词`hotfix`)直接推向生产。

但是特征分支呢？在这篇博客文章中，我详细分析了什么是特性分支，以及如何在 Octopus 中管理它们。

## 什么是特征分支？

在我们能够在 Octopus 中建模特征分支之前，我们需要理解什么是特征分支。

[马丁·福勒提供了很好的描述](https://martinfowler.com/bliki/FeatureBranch.html):

> 特性分支是一种源代码分支模式，当开发人员开始处理一个新特性时，她会打开一个分支。她在这个分支上完成特性的所有工作，并在特性完成时将变更与团队的其他成员集成在一起。

我怀疑大多数开发人员都熟悉在一个特性分支中工作，但是我们感兴趣的是与部署相关的细节，包含在短语*在这个分支*上做所有的特性工作。

具体来说，我们希望为开发人员提供一种部署他们正在开发的代码的方式:

*   在不容易在本地复制的环境中进行测试，例如平台即服务(PaaS)产品。
*   轻松地与团队的其他成员分享他们工作的当前状态。
*   为额外的测试提供稳定的目标，如端到端、性能或安全性测试。

不像固定的环境，比如测试或者生产，特性分支是短暂的。它们在一个特性被开发时存在，但是一旦合并到一个主线分支中，这个特性分支就应该被删除。

此外，功能分支并不打算部署到生产中。与热修复不同，热修复是快速解决关键问题的紧急生产部署，功能分支仅用于测试。

通过限制某个功能分支的受众，您还可以潜在地节省成本。这是因为您可以在晚上删除部署及其底层基础设施，然后在早上重新部署特性分支。这样做意味着你不再花钱去托管那些一夜之间没人会用到的应用程序。

特性分支通常由 CI 系统处理，这是一种确保测试通过并产生可部署工件的便捷方式。有一个令人信服的论点是，CI 系统应该产生一个可部署的工件(如果代码编译的话),而不管测试结果如何，假设像测试驱动开发(TDD)这样的过程鼓励失败的测试作为开发工作流的正常部分。

### 版本控制功能分支工件

像 GitVersion 这样的工具提供了一些例子，展示了包含在 SemVer 预发布字段中的[特性分支名称，产生了类似于`1.3.0-myfeature`的版本。大多数包管理工具都有公开组件以适应特性分支名称的版本控制策略。](https://gitversion.net/docs/learn/branching-strategies/gitflow/examples)

在包管理器没有版本控制指南的情况下，比如 Docker 存储库，采用 SemVer 这样的版本控制方案是一个不错的选择。

Maven 版本控制方案有一个怪癖，带有限定符的版本，如`1.0.0-myfeature`，被认为是比未限定的版本，如`1.0.0`更晚的版本。但是，使用通道版本规则意味着代表功能分支的合格版本不适合部署到生产环境，因此合格和不合格版本的排序在 Octopus 中不存在问题。

### 功能分支部署的质量

考虑到以上所有因素，我们可以将功能分支部署定义为具有以下品质:

*   它们仅用于测试，不会在生产环境中公开。
*   它们是短暂的，仅在版本控制系统中存在相应的分支时才存在。
*   在工作时间之外可能不需要它们，如果部署可以在一夜之间停止，则可以节省成本。
*   它们不会在任何地方得到提升，因此它们的生命周期包括在测试环境中的一次部署。
*   他们的应用程序工件被版本化以识别源特性分支。

下一步是在 Octopus 中对上述规则进行建模。

Octopus 包含了有限的元步骤概念，我们将使用这个术语对修改 Octopus 本身的步骤进行分类。

**Deploy a release** 步骤就是一个例子，顾名思义，它可以用于部署为另一个项目创建的发布。[脚本步骤还可以动态生成一些章鱼资源](https://octopus.com/docs/infrastructure/deployment-targets/dynamic-infrastructure)。

元步骤功能有些特别和有限。我们需要一个全面的解决方案来创建和销毁 Octopus 中的资源，以反映功能分支的短暂性质。

Octopus CLI 提供了一些额外的功能，能够创建许多 Octopus 资源，如版本、频道和环境。不幸的是，它不包括删除这些资源的所有相应选项，所以它不是一个完整的解决方案。

另一种选择是使用 REST API，它公开了 web UI 可以执行的每个操作。然而，如果可能的话，我宁愿避免编写第二个 CLI 工具来管理 Octopus 资源的整个生命周期。

幸运的是，[章鱼平台供应商](https://registry.terraform.io/providers/OctopusDeployLabs/octopusdeploy/latest/docs)正好提供了我们需要的东西。将这个提供者与 Octopus 中现有的 Terraform 部署步骤结合起来，我们可以创建自己的元步骤。这反过来意味着我们可以管理表示特征分支所需的短暂 Octopus 资源。

## 管理功能分支的操作手册

我们将利用 runbooks 来支持创建和删除短暂的 Octopus 资源。这允许我们将底层基础设施的管理与应用程序的部署分开。

我们将创建六本操作手册。三个 runbooks 将为单个功能分支创建、删除和暂停资源。根据 Git 中是否存在分支，这些将与另外三个 runbooks 配对执行:

*   创建分支机构基础设施，这将创建部署单个分支机构所需的资源。
*   恢复分支，这将基于 Git 存储库中的分支基础设施(重新)创建分支基础设施。
*   破坏分支基础设施，这将破坏特征分支资源。
*   销毁分支，这将删除 Git 中删除的分支资源的任何特性分支资源。
*   暂停分支基础设施，这将在不使用时关闭或破坏昂贵的功能分支资源。
*   挂起分支，这将挂起所有分支。

## 将章鱼资源映射到特色分支

Octopus 有几个资源来促进功能分支的部署，但是对于这些资源如何与功能分支相关联，并没有硬性的规则。我们将在这里提供一些意见，指导我们如何构建上面列出的操作手册。

### 环境

我们将使用环境来封装用于托管特性分支的资源。环境提供了一个很好的范围和安全边界，并且清楚地将特性分支与静态环境(如生产环境)分开。

### 生活过程

为了防止特性分支被提升到生产环境，或者根本不被提升，我们将为每个特性分支创建一个生命周期，包括上面的单个环境。

### 频道

为了确保适当的特性分支工件被部署到正确的环境中，我们将使用一个通道。通道规则将确保特性分支工件版本被选择并导向正确的生命周期，这反过来确保正确的环境接收特性分支部署。

在这三种资源中，我们将通道视为特性分支的表示。您在部署期间选择通道，它是 Octopus 项目的唯一本地资源(生命周期和环境的范围是一个空间)。这使得它成为在项目中表现一个特性分支的自然选择。

租户也可以用来表示一个功能分支。我发现环境、渠道和生命周期是更自然的匹配。

我们部署到的 web 应用程序将由目标来表示。这些目标将被限定在它们相关的功能分支环境中。

Web 应用程序有插槽的概念，插槽也可以用来部署特性分支。然而，插槽是 Azure 服务计划上的有限资源，这意味着您将受限于任何时候可以部署的功能分支的数量。因此，本博客为每个功能分支使用了新的网络应用。

## 最初的 Azure 帐户

为了在 Azure 中部署功能分支 web 应用，我们需要一个初始的“种子”Azure 帐户来管理云资源。这是运行 Terraform 脚本的帐户:

[![](img/da0a227144c73b613c9bbe2541cab005.png)](#)

我们的元步骤将像任何其他 Terraform 部署一样执行。这意味着即使 Octopus 在管理自己，我们也需要创建变量来允许 Terraform 部署返回到 Octopus 服务器。

1.  创建一个新的 Octopus 项目，并定义两个名为 **ApiKey** 和 **ServerUrl** 的变量。API 密钥将是一个保存有 [Octopus API 密钥](https://octopus.com/docs/octopus-rest-api/how-to-create-an-api-key)的秘密变量，服务器 URL 将是 Octopus 服务器的 URL。在这个例子中，我使用了一个 URL 为`https://mattc.octopus.app`的[云实例](https://octopus.com/docs/octopus-cloud)。
2.  此外，创建一个名为 **FeatureBranch** 的提示变量。这将用于将功能分支的名称传递到 runbooks 中。
3.  最后，创建一个名为 **Azure** 的变量，该变量指向上一节中创建的 Azure 帐户:

【T2 ![](img/924afa5e2425aed9da3e9e234840593f.png)

## 部署项目

我们的部署项目将利用**部署 Azure 应用服务**步骤从**octopussamples/randomquotes Java**Docker 存储库中部署 Docker 映像。[存储库](https://hub.docker.com/r/octopussamples/randomquotesjava)包含使用 GitHub 上的[代码构建的图像，带有正确的标记规则以识别特征分支。](https://Github.com/OctopusSamples/RandomQuotes-Java)

称该步骤为`Deploy Web App`。我们将在下一节创建通道时引用这个步骤名。

## 《创建分支机构基础设施操作手册》

我们将从创建构建所有资源的 runbook 开始，包括 Octopus 和 Azure，以部署一个特性分支。

这个 Terraform 模板创建了 Azure web 应用基础设施。我们将它保存在一个单独的模板中，这样我们可以单独销毁这些资源，而不会破坏 Octopus 资源。该模板将被部署在名为 **Feature Branch Web App** 的步骤中(我们稍后将使用步骤名称来引用输出变量):

```
provider "azurerm" {
version = "=2.0.0"
features {}
}

variable "apiKey" {
    type = string
}

variable "space" {
    type = string
}

variable "serverURL" {
    type = string
}

terraform {
  backend "azurerm" {
    resource_group_name  = "mattc-test"
    storage_account_name = "storageaccountmattc8e9b"
    container_name       = "terraformstate"
    key                  = "#{FeatureBranch}.azure.terraform.tfstate"
  }
}

resource "azurerm_resource_group" "resourcegroup" {
  name     = "mattc-test-webapp-#{FeatureBranch}"
  location = "West Europe"
  tags = {
    OwnerContact = "@matthew.casperson"
    Environment = "Dev"
    Lifetime = "15/5/2021"
  }
}

resource "azurerm_app_service_plan" "serviceplan" {
  name                = "#{FeatureBranch}"
  location            = azurerm_resource_group.resourcegroup.location
  resource_group_name = azurerm_resource_group.resourcegroup.name
  kind                = "Linux"
  reserved            = true

  sku {
    tier = "Standard"
    size = "S1"
  }
}

resource "azurerm_app_service" "webapp" {
  name                = "mattctestapp-#{FeatureBranch}"
  location            = azurerm_resource_group.resourcegroup.location
  resource_group_name = azurerm_resource_group.resourcegroup.name
  app_service_plan_id = azurerm_app_service_plan.serviceplan.id
}

output "ResourceGroupName" {
  value = "${azurerm_resource_group.resourcegroup.name}"
}

output "WebAppName" {
  value = "${azurerm_app_service.webapp.name}"
} 
```

下一个 Terraform 模板创建了一个 Octopus 环境，将其置于生命周期中，并在通道中对其进行配置。它还创建了一个 web 应用程序目标:

```
variable "apiKey" {
    type = string
}

variable "space" {
    type = string
}

variable "serverURL" {
    type = string
}

terraform {
  required_providers {
    octopusdeploy = {
      source  = "OctopusDeployLabs/octopusdeploy"
    }
  }

  backend "azurerm" {
    resource_group_name  = "mattc-test"
    storage_account_name = "storageaccountmattc8e9b"
    container_name       = "terraformstate"
    key                  = "#{FeatureBranch}.terraform.tfstate"
  }
}

provider "octopusdeploy" {
  address  = var.serverURL
  api_key   = var.apiKey
  space_id = var.space
}

resource "octopusdeploy_environment" "environment" {
  allow_dynamic_infrastructure = true
  description                  = "Feature branch environment for #{FeatureBranch}"
  name                         = "#{FeatureBranch}"
  use_guided_failure           = false
}

resource "octopusdeploy_lifecycle" "lifecycle" {
  description = "The lifecycle holding the feature branch #{FeatureBranch}"
  name        = "#{FeatureBranch}"

  release_retention_policy {
    quantity_to_keep    = 1
    should_keep_forever = true
    unit                = "Days"
  }

  tentacle_retention_policy {
    quantity_to_keep    = 30
    should_keep_forever = false
    unit                = "Items"
  }

  phase {
    name                        = "#{FeatureBranch}"
    is_optional_phase           = false
    optional_deployment_targets = ["${octopusdeploy_environment.environment.id}"]

    release_retention_policy {
      quantity_to_keep    = 1
      should_keep_forever = true
      unit                = "Days"
    }

    tentacle_retention_policy {
      quantity_to_keep    = 30
      should_keep_forever = false
      unit                = "Items"
    }
  }
}

resource "octopusdeploy_channel" "channel" {
  name       = "Feature branch #{FeatureBranch}"
  project_id = "#{Octopus.Project.Id}"
  lifecycle_id = "${octopusdeploy_lifecycle.lifecycle.id}"
  description = "Repo: https://Github.com/OctopusSamples/RandomQuotes-Java Branch: #{FeatureBranch}"
  rule {
    id = "#{FeatureBranch}"
    tag = "^#{FeatureBranch}.*$"
    action_package {
      deployment_action = "Deploy Web App"
    }
  }
}

resource "octopusdeploy_azure_service_principal" "azure_service_principal_account" {
  application_id  = "#{Azure.Client}"
  name            = "Azure account for #{FeatureBranch}"
  password        = "#{Azure.Password}"
  subscription_id = "#{Azure.SubscriptionNumber}"
  tenant_id       = "#{Azure.TenantId}"
}

resource "octopusdeploy_azure_web_app_deployment_target" "webapp" {
  account_id                        = "${octopusdeploy_azure_service_principal.azure_service_principal_account.id}"
  name                              = "Azure Web app #{FeatureBranch}"
  resource_group_name               = "#{Octopus.Action[Feature Branch Web App].Output.TerraformValueOutputs[ResourceGroupName]}"
  roles                             = ["Web Application", "Web Application #{FeatureBranch}"]
  tenanted_deployment_participation = "Untenanted"
  web_app_name                      = "#{Octopus.Action[Feature Branch Web App].Output.TerraformValueOutputs[WebAppName]}"
  environments                      = ["${octopusdeploy_environment.environment.id}"]
  endpoint {
    default_worker_pool_id          = "WorkerPools-762"
    communication_style             = "None"
  }
} 
```

在这个模板中有一些重要的东西需要强调。

Octopus provider 是一个官方插件，可以通过 Terraform 自动下载:

```
 required_providers {
    octopusdeploy = {
      source  = "OctopusDeployLabs/octopusdeploy"
    }
  } 
```

我们已经利用 Octopus 模板语法(后跟花括号的散列符号)构建了 Terraform 模板的各个部分。后端块[不能使用 Terraform 变量](https://www.terraform.io/docs/language/settings/backends/configuration.html#using-a-backend-block)，但是因为 Octopus 在将文件传递给 Terraform 之前对其进行了处理，所以我们可以绕过这一限制，将变量注入到键名中:

```
 backend "azurerm" {
    resource_group_name  = "mattc-test"
    storage_account_name = "storageaccountmattc8e9b"
    container_name       = "terraformstate"
    key                  = "#{FeatureBranch}.terraform.tfstate"
  } 
```

对通道的描述指出了通道所代表的 Git 存储库和分支。因为我们决定通道是项目中一个特性分支的表示，所以将它链接回它所来自的代码库所需的细节将在这里被捕获。

还要注意，我们引用前面创建的步骤`Deploy Web App`来将通道规则应用于:

```
resource "octopusdeploy_channel" "channel" {
  name       = "Feature branch #{FeatureBranch}"
  project_id = "#{Octopus.Project.Id}"
  lifecycle_id = "${octopusdeploy_lifecycle.lifecycle.id}"
  description = "Repo: https://Github.com/OctopusSamples/RandomQuotes-Java Branch: #{FeatureBranch}"
  rule {
    id = "#{FeatureBranch}"
    tag = "^#{FeatureBranch}.*$"
    action_package {
      deployment_action = "Deploy Web App"
    }
  } 
```

为了创建 Azure 帐户，我们创建了一个新帐户，其详细信息与在 Terraform 步骤中定义的 Azure 帐户相同:

[![](img/0b73eec3e5fc46bd1c557b4bf56ef136.png)](#)

在一个更安全的环境中，这些细节将被一个仅限于必要资源的帐户所取代。在不太安全的环境中，您可以完全跳过创建功能分支特定帐户，而只需共享现有帐户:

```
resource "octopusdeploy_azure_service_principal" "azure_service_principal_account" {
  application_id  = "#{Azure.Client}"
  name            = "Azure account for #{FeatureBranch}"
  password        = "#{Azure.Password}"
  subscription_id = "#{Azure.SubscriptionNumber}"
  tenant_id       = "#{Azure.TenantId}"
} 
```

当创建目标时，我们利用由第一个 Terraform 脚本创建的输出变量。资源组名输出为`#{Octopus.Action[Feature Branch Web App].Output.TerraformValueOutputs[ResourceGroupName]}`，web app 名输出为`#{Octopus.Action[Feature Branch Web App].Output.TerraformValueOutputs[WebAppName]}`。

这个目标还专门为健康检查定义了一个 Windows worker 池(在我的例子中，ID 为`WorkerPools-762`)。这是因为 Azure web 应用目标需要 Windows 工作线程来运行运行状况检查:

```
resource "octopusdeploy_azure_web_app_deployment_target" "webapp" {
  account_id                        = "${octopusdeploy_azure_service_principal.azure_service_principal_account.id}"
  name                              = "Azure Web app #{FeatureBranch}"
  resource_group_name               = "#{Octopus.Action[Feature Branch Web App].Output.TerraformValueOutputs[ResourceGroupName]}"
  roles                             = ["Web Application", "Web Application #{FeatureBranch}"]
  tenanted_deployment_participation = "Untenanted"
  web_app_name                      = "#{Octopus.Action[Feature Branch Web App].Output.TerraformValueOutputs[WebAppName]}"
  environments                      = ["${octopusdeploy_environment.environment.id}"]
  endpoint {
    default_worker_pool_id          = "WorkerPools-762"
    communication_style             = "None"
  } 
```

本操作手册的最后一步是调用 Octopus CLI 来创建一个版本并将其部署到新环境中。通过运行部署，我们可以确信，当我们在早上重新创建特性分支时，它们将部署最新的应用程序代码。

如果已经创建了一个分支，但是没有可用于部署的工件，我们允许这个调用失败:

```
octo create-release \
    --project="Random Quotes" \
    --channel="Feature branch #{FeatureBranch}" \
    --deployTo="#{FeatureBranch}" \
    --space="#{Octopus.Space.Name}" \
    --server="#{ServerUrl}" \
    --apiKey="#{ApiKey}"

# allow failure if the a package is not available
exit 0 
```

通过运行本操作手册:

*   Octopus 将填充所有的功能分支资源。
*   Azure 中创建了一个 web 应用程序。
*   部署最新版本的功能分支应用程序(如果存在)。

因为 Terraform 是等幂的，所以我们可以多次重新运行这个 runbook，任何丢失的资源都将被重新创建。我们将利用这一点，在一夜之间销毁网络应用程序，以节省成本。

## 简历分支运行手册

Create Branch infra structure run book 可以为分支创建资源，但是它不知道 Git 中存在哪些分支。 **Resume Branches** runbook 扫描 Git repo 以查找分支，并执行**Create Branch infra structure**run book 以在 Octopus 中反映这些分支。

下面的 PowerShell 解析对`ls-remote --heads https://repo`的调用结果，并使用结果列表来调用**创建分支基础结构** runbook:

```
$octopusURL = "https://mattc.octopus.app"
$octopusAPIKey = "#{ApiKey}"
$spaceName = "#{Octopus.Space.Name}"
$projectName = "#{Octopus.Project.Name}"
$environmentName = "#{Octopus.Environment.Name}"
$repos = @("https://Github.com/OctopusSamples/RandomQuotes-Java.Git")
$ignoredBranches = @("master", "main")

# Get all the branches for the repos listed in the channel descriptions    
$branches = $repos |
    % {PortableGit\bin\Git ls-remote --heads $_} |
    % {[regex]::Match($_, "\S+\s+(\S+)").captures.groups[1].Value} |
    % {$_.Replace("refs/heads/", "")} |
    ? {-not $ignoredBranches.Contains($_)}

# Clean up the old branch infrastructure
$branches |
    % {Write-Host "Creating branch $_";
        octo run-runbook `
          --apiKey $octopusAPIKey `
          --server $octopusURL `
          --project $projectName `
          --runbook "Create Branch Infrastructure" `
          -v "FeatureBranch=$_" `
          --environment $environmentName `
          --space $spaceName} 
```

## 销毁分支机构基础设施操作手册

**销毁分支基础设施**与**创建分支基础设施**相反，它删除所有资源，而不是创建它们。这是通过使用与在**创建分支基础设施**操作手册中定义的相同的 Terraform 模板来实现的，但是使用**销毁 Terraform 资源**步骤而不是**应用 Terraform 模板**步骤来执行它们。

## 摧毁树枝操作手册

**破坏分支**运行手册与**恢复分支**运行手册相反。它向 Octopus 查询所有通道，向 Git 查询所有分支，并且在通道没有匹配分支的情况下，调用**销毁分支基础结构** runbook:

```
Install-Module -Force PowershellOctopusClient
Import-Module PowershellOctopusClient

$channelDescriptionRegex = "Repo:\s*(\S+)\s*Branch:\s*(\S+)"

# Octopus variables
$octopusURL = "#{OctopusUrl}"
$octopusAPIKey = "#{ApiKey}"
$spaceName = "#{Octopus.Space.Name}"
$projectName = "#{Octopus.Project.Name}"

$endpoint = New-Object Octopus.Client.OctopusServerEndpoint $octopusURL, $octopusAPIKey
$repository = New-Object Octopus.Client.OctopusRepository $endpoint
$client = New-Object Octopus.Client.OctopusClient $endpoint

$space = $repository.Spaces.FindByName($spaceName)
$repositoryForSpace = $client.ForSpace($space)

$project = $repositoryForSpace.Projects.FindByName($projectName)
$channels = $repositoryForSpace.Channels.GetAll() | ? {$_.ProjectId -eq $project.Id}

# Get all the project channels whose description is in the format "Repo: http://blah Branch: blah"
$repos = $channels |
    % {[regex]::Match($_.Description, $channelDescriptionRegex)} |
    ? {$_.Success} |
    % {$_.captures.groups[1]} |
    Get-Unique

# Get all the branches for the repos listed in the channel descriptions    
$branches = $repos |
    % {@{Repo = $_; Branch = $(PortableGit\bin\Git ls-remote --heads $_)}} |
    % {$repo = $_.Repo; [regex]::Match($_.Branch, "\S+\s+(\S+)").captures.groups[1].Value.Replace("refs/heads/", "") | %{"Repo: $repo Branch: $_"}}

# Get all the branches that have a channel but have been removed from the repo
$oldChannels = $channels |
    ? {[regex]::Match($_.Description, $channelDescriptionRegex).Success} |
    ? {-not $branches.Contains($_.Description)} |
    % {[regex]::Match($_.Description, $channelDescriptionRegex).captures.groups[2]}

# Clean up the old branch infrastructure
$oldChannels |
    % {Write-Host "Deleting branch $_";
        octo run-runbook `
          --apiKey $octopusAPIKey `
          --server $octopusURL `
          --project $projectName `
          --runbook "Destroy Branch Infrastructure" `
          -v "FeatureBranch=$_" `
          --environment $_ `
          --space $spaceName} 
```

## 暂停分支机构基础设施运行手册

为了一夜之间节省资金，我们将销毁 Azure 中的功能分支 web 应用。这是微软提出的[暂停应用服务计划的解决方案。](https://feedback.azure.com/forums/169385-web-apps/suggestions/13550325-ability-to-suspend-app-service-plans-without-charg)

删除 Azure 资源也只是运行**销毁 Terraform 资源**步骤的一个例子，模板在**创建分支基础设施**操作手册中的**功能分支 Web 应用**步骤中。因为我们只破坏 Azure 资源，所以任何现有的 Octopus 版本、渠道、环境、目标和帐户都将保留。

由于我们在删除和创建这些 Azure 资源时使用相同的后端状态文件，Terraform 知道任何给定 Azure 资源的状态。这意味着我们可以根据需要调用**暂停分支基础设施**、**创建分支基础设施**、**销毁分支基础设施**run book，Terraform 将让 Azure 处于所需的状态。

## 暂停分支运行手册

**暂停分支**运行手册扫描 Octopus 寻找通道，并调用**暂停分支基础设施**运行手册。这意味着任何与渠道相关的 Azure 资源，以及 Octopus 中的功能分支都将被销毁，并且不再需要花钱:

```
Install-Module -Force PowershellOctopusClient
Import-Module PowershellOctopusClient

$channelDescriptionRegex = "Repo:\s*(\S+)\s*Branch:\s*(\S+)"

# Octopus variables
$octopusURL = "#{ServerUrl}"
$octopusAPIKey = "#{ApiKey}"
$spaceName = "#{Octopus.Space.Name}"
$projectName = "#{Octopus.Project.Name}"
$environment = "#{Octopus.Environment.Name}"

$endpoint = New-Object Octopus.Client.OctopusServerEndpoint $octopusURL, $octopusAPIKey
$repository = New-Object Octopus.Client.OctopusRepository $endpoint
$client = New-Object Octopus.Client.OctopusClient $endpoint

$space = $repository.Spaces.FindByName($spaceName)
$repositoryForSpace = $client.ForSpace($space)

$project = $repositoryForSpace.Projects.FindByName($projectName)
$channels = $repositoryForSpace.Channels.GetAll() | ? {$_.ProjectId -eq $project.Id}

# Get all the project channels whose description is in the format "Repo: http://blah Branch: blah"
$branches = $channels |
    % {[regex]::Match($_.Description, $channelDescriptionRegex)} |
    ? {$_.Success} |
    % {$_.captures.groups[2].Value} |
    Get-Unique

# Clean up the old branch infrastructure
$branches |
    % {Write-Host "Deleting branch $_";
        octo run-runbook `
          --apiKey $octopusAPIKey `
          --server $octopusURL `
          --project $projectName `
          --runbook "Suspend Branch Infrastructure" `
          -v "FeatureBranch=$_" `
          --environment $environment `
          --space $spaceName} 
```

## 同步 Octopus 和 Git

我们现在有了管理功能分支资源所需的所有操作手册。为了保持 Octopus 和 Git 同步，下一步是调度触发器来删除旧的分支、关闭资源和创建新的分支。

我们首先在一天中的不同时间点触发**销毁分支**操作手册。该 runbook 将破坏功能分支资源，其中底层功能分支已从 Git 中删除。

清理这些分支不应该干扰任何人，因为我们假设删除底层分支意味着该特性已经被合并或放弃。所以我们创建一个触发器，每小时执行一次**销毁分支**运行手册。

同样，我们希望捕捉全天创建的任何新分支，因此我们安排 **Create Branches** runbook 在早上 6 点到下午 6 点之间定期运行，以覆盖平均工作日。

为了降低成本，我们希望在一天结束时删除托管我们的功能分支的云资源。为此，我们安排**暂停分支机构**运行手册在晚上 7 点运行。这确保它在最终的**创建分支** runbook 触发器之后运行。

最终结果是:

*   通过**销毁分支**，全天清理被删除的分支。
*   在工作日开始时，任何新的分支都会被检测到，所有 Azure 资源都会通过**创建分支**重新创建。
*   晚上所有 Azure 资源都被**暂停分支**破坏:

[![](img/039b1e536ded5e0771877bf2fc23db60.png)](#)

如果使用触发器的响应能力不足以捕捉新分支的创建和销毁，也可以直接从 Git 本身执行`octo run-runbook`。GitHub 等流行的托管服务提供工具，允许在创建和销毁分支时运行脚本，这使得 Octopus 中特性分支的创建和清理几乎是即时的。

## 看章鱼的特征分支

有了这些操作手册并适当触发后，我们现在可以回顾在 Octopus 中创建的资源来表示我们的特性分支。

以下是渠道:

[![](img/7300ce57b8e0d7a4fd4aa7ce2681aa8a.png)](#)

以下是环境:

[![](img/0e6e9d6a152166e2d2a8f6774a52091e.png)](#)

以下是生命周期:

[![](img/b431a3d59d765c63210904743c7ff467.png)](#)

以下是目标:

[![](img/ea7734baded7431bfc247a0ccdccb5ba.png)](#)

这些是账目:

[![](img/dec9ef419b89937a53330a06493a55b1.png)](#)

这里是发布创建，我们选择一个要部署到的通道/特性分支，并让 Octopus 匹配通道规则来为我们选择正确的特性分支工件:

[![](img/89eac7ebe75c24ae2b63e80e3eab8bc6.png)](#)

## 结论

Terraform 应用和销毁步骤，与 Octopus Terraform 提供程序一起，为我们提供了创建元步骤以在 Octopus 中实现短期特性分支所需的工具。由于 Terraform 的幂等性质，我们有一组可靠地管理我们短暂的 Azure 和 Octopus 资源的健壮步骤。

通过一些自定义脚本来同步 Octopus 与 Git 分支和调度触发器或来自 GitHub 等托管平台的直接触发器，我们可以确保 Octopus 反映我们代码库中正在开发的功能分支。

## 观看网络研讨会

[https://www.youtube.com/embed/NRwFdpvNYyA](https://www.youtube.com/embed/NRwFdpvNYyA)

VIDEO

我们定期举办网络研讨会。请参见[网络研讨会第](https://octopus.com/events)页，了解过去的网络研讨会以及即将举办的网络研讨会的详细信息。

愉快的部署！