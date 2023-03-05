# 从 SSISDB 导入变量- Octopus Deploy

> 原文：<https://octopus.com/blog/get-variables-from-ssisdb>

## 介绍

如果您曾经使用过[从包](https://library.octopus.com/step-templates/bf005449-60c2-4746-8e07-8ba857f93605/actiontemplate-deploy-ispac-ssis-project-from-a-package)中部署 ispac SSIS 项目或[从引用的包](https://library.octopus.com/step-templates/0c8167e9-49fe-4f2a-a007-df5ef2e63fac/actiontemplate-deploy-ispac-ssis-project-from-referenced-package)中部署 ispac SSIS 项目步骤模板，那么您会知道它可以从您的 SSIS 包中提取项目参数和连接管理器信息，并在 SSISDB 中将它们创建为环境变量。您还知道，在执行此操作时，连接管理器的每个属性都被创建为一个单独的环境变量。如果您的 Octopus 项目中没有同名的变量，您会得到一条消息:

OctopusParameters 集合为空或 CM。OctoDemoSql . adventureworks 2017 . sa . connectusingmanagedidentity 不在集合中

该变量仍在 SSISDB 环境中创建，但是，它默认为设计时值。如果您的 SSIS 包有大量的项目参数和/或连接管理器，那么变量列表会非常庞大，老实说，一个一个地创建是非常乏味的。

## 自动化拯救世界！

作为从一个包部署 ispac SSIS 项目的最初作者，我可以告诉你，我的 SSIS 开发人员拿着干草叉和火把敲我办公室的门，咆哮着说创建所有这些变量是多么耗时。为了避免死于他们眼中的匕首，我求助于 PowerShell 和 Octopus Deploy API，想出一种方法来从 SSISDB 环境中检索变量，并将它们导入到他们的 Octopus Deploy 项目中。

### 剧本

以下脚本用于演示目的。

下面的脚本从 SSISDB 环境中提取变量和值，并在 Octopus Deploy 中将它们创建为项目变量！这节省了大量的时间，开发人员满意地离开了我的办公室，他们的要求得到了满足。

```
# Define parameters
param(
    $OctopusServerUrl,
    $APIKey
)

# Define functions
Function Get-EnvironmentVariablesFromSSISDB
{
    # Define parameters
    param(
        $UseIntegratedAuthentication,
        $SqlUserName,
        $SqlPassword,
        $CatalogName,
        $FolderName,
        $EnvironmentName,
        $SqlServerName
    )

    # Import needed assemblies
    [Reflection.Assembly]::LoadWithPartialName("Microsoft.SqlServer.Management.IntegrationServices") | Out-Null # Out-Null supresses a message that would normally be displayed saying it loaded out of GAC

    # Create a connection to the server
    $sqlConnectionString = "Data Source=$SqlServerName;Initial Catalog=master;"

    # Check authentication
    if ($UseIntegratedAuthentication)
    {
        # Add integrated
        $sqlConnectionString += "Integrated Security=SSPI;"
    }
    else
    {
        # ass username password
        $sqlConnectionString += "User ID=$SqlUserName; Password=$SqlPassword"    
    }

    $sqlConnection = New-Object System.Data.SqlClient.SqlConnection $sqlConnectionString

    # create integration services object
    $integrationServices = New-Object "$ISNamespace.IntegrationServices" $sqlConnection

    try
    {
        # get catalog reference
        $Catalog = Get-Catalog -CatalogName $CataLogName -IntegrationServices $integrationServices
        $Folder = Get-Folder -FolderName $FolderName -Catalog $Catalog
        $Environment = Get-Environment -Folder $Folder -EnvironmentName $EnvironmentName

        # return environment variables
        return $Environment.Variables
    }
    finally
    {
        # close connection
        $sqlConnection.Close()
    }
}

Function Get-Folder
{
    # parameters
    Param($FolderName, $Catalog)

    # try to get reference to folder
    $Folder = $Catalog.Folders[$FolderName]

    # check to see if $Folder has a value
    if(!$Folder)
    {
        Write-Error "Folder not found."
        throw
    }

    # return the folde reference
    return $Folder
}

Function Get-Environment
{
    # define parameters
    Param($Folder, $EnvironmentName)

    # get reference to Environment
    $Environment = $Folder.Environments[$EnvironmentName]

    # check to see if it's a null reference
    if(!$Environment)
    {
        Write-Error "Environment not found."
        throw
    }

    # return the environment
    return $Environment
}

Function Get-Catalog
{
    # define parameters
    Param ($CatalogName, $IntegrationServices)

    # define working varaibles
    $Catalog = $null

    # check to see if there are any catalogs
    if($integrationServices.Catalogs.Count -gt 0 -and $integrationServices.Catalogs[$CatalogName])
    {
        # get reference to catalog
        $Catalog = $integrationServices.Catalogs[$CatalogName]
    }
    else
    {
        Write-Error  "Catalog $CataLogName does not exist or the Tentacle account does not have access to it."

        # throw error
        throw
    }

    # return the catalog
    return $Catalog
}

Function Get-OctopusProject
{
    # Define parameters
    param(
        $OctopusServerUrl,
        $ApiKey,
        $ProjectName
    )

    # Call API to get all projects, then filter on name
    $octopusProject = Invoke-RestMethod -Method "get" -Uri "$OctopusServerUrl/api/projects/all" -Headers @{"X-Octopus-ApiKey"="$ApiKey"}

    # return the specific project
    return ($octopusProject | Where-Object {$_.Name -eq $ProjectName})
}

Function Get-OctopusProjectVariables
{
    # Define parameters
    param(
        $OctopusDeployProject,
        $OctopusServerUrl,
        $ApiKey
    )

    # Get reference to the variable list
    return (Invoke-RestMethod -Method "get" -Uri "$OctopusServerUrl/api/variables/$($OctopusDeployProject.VariableSetId)" -Headers @{"X-Octopus-ApiKey"="$ApiKey"})
}

Function Update-ProjectVariables
{
    param(
        $OctopusServerUrl,
        $ProjectVariables,
        $ApiKey
    )

    # Convert the object into JSON
    $jsonBody = $ProjectVariables | ConvertTo-Json -Depth 5

    # Call the API to update
    Invoke-RestMethod -Method "put" -Uri "$OctopusServerUrl/api/variables/$($ProjectVariables.Id)" -Body $jsonBody -Headers @{"X-Octopus-ApiKey"="$ApiKey"}
}

try
{
    # Store the IntegrationServices Assembly namespace to avoid typing it every time
    $ISNamespace = "Microsoft.SqlServer.Management.IntegrationServices"
    $CataLogName = "SSISDB"

    # Get reference to project
    $octopusProject = Get-OctopusProject -OctopusServerUrl "<Your URL Here>" -ApiKey "<Your API key here>" -ProjectName "<Octopus Deploy project name>"

    # Get list of existing variables
    $octopusProjectVariables = Get-OctopusProjectVariables -OctopusDeployProject $octopusProject -OctopusServerUrl "<Your URL Here>" -ApiKey "<Your API key here>"

    # Get list of SSIS project variables
    $ssisEnvironmentVariables = Get-EnvironmentVariablesFromSSISDB -UseIntegratedAuthentication $false -SqlServerName "<Sql server name>" -SqlUserName "<sql account user name>" -SqlPassword "<sql account password>" -CatalogName "SSISDB" -FolderName "<SSISDB folder name>" -EnvironmentName "<SSISDB environment name>"

    # Loop through the ssis variable set
    foreach ($variable in $ssisEnvironmentVariables)
    {
        # Check to see if variable already exists in Octopus project variables
        if ($null -eq ($octopusProjectVariables.Variables | Where-Object {$_.Name -eq $variable.Name}))
        {
            # Display message
            Write-Output "Adding $($variable.Name) to Octopus Deploy project $($octopusProject.Name)"

            # Create new variable hash table
            $newVariable = @{
                #Id = "$(New-Guid)"
                Name = "$($variable.Name)"
                Value = "$($variable.Value)"
                Description = $null
                Scope = @{}
                IsEditable = $(if ($variable.Sensitive) { $false} else {$true})
                Prompt = $null
                Type = "String"
                IsSensitive = $(if ($variable.Sensitive) { $true} else {$false})
            }

            # Add variable
            $octopusProjectVariables.Variables += $newVariable
        }
    }

    # Update the project
    Update-ProjectVariables -ProjectVariables $octopusProjectVariables -ApiKey $APIKey -OctopusServerUrl $OctopusServerUrl
}
catch
{
    Write-Error $_.Exception.Message

    throw
} 
```

## 摘要

在本文中，我向您展示了一种快速填充 Octopus Deploy 项目变量的方法，方法是连接到 SSISDB 并复制环境变量。该解决方案要求至少部署一次 SSIS 包，以便在 SSISDB 中填充环境变量。虽然这个例子是特定于 SSISDB 的，但是使用 API 以编程方式向 Octopus Deploy 项目添加变量的一般方法可以用于各种各样的源。