# 使用 JSON - Octopus Deploy 验证应用程序设置

> 原文：<https://octopus.com/blog/verify-appsettings-json>

[![Illustration showing Octopus variables being scanned](img/88bc01de9e8e0d4f1f421987941f5022.png)](#)

在我的[上一篇文章](https://octopus.com/blog/verify-appsettings-or-variable-replacement)中，我展示了如何验证 web.config 文件中的所有应用程序设置都有相应的 Octopus 项目变量，以及如何确保所有为变量替换而配置的文件不会留下任何占位符。这篇文章将重点验证存储在 JSON 配置文件中的应用程序设置。在这种情况下我们看到的是。NET Core appSettings.json 文件，但它可以扩展到任何 json 配置文件。

## 比较环境文件

像 appSettings . development . JSON 这样的. NET 核心应用程序的环境应用程序设置文件并不少见。Deveopment.json 文件，也需要将其添加到 appSettings.json 文件中。为了解决这个问题，我们可以创建一个带有递归调用的 PowerShell 函数来遍历 JSON 文件键，然后将条目与环境文件进行比较。与上一篇文章类似，我们可以使用`$settingsToIgnore`数组定义我们想要忽略的任何设置:

```
Function Get-AppSettings
{
    # Define parameters
    param($jsonObject,
        $parentNamespace,
        $settingsToIgnore)

    $namespace = ""
    $tempArray = @()

    if (![string]::IsNullOrEmpty($jsonObject) -and ($jsonObject.GetType().Name -eq "PSCustomObject"))
    {

        # Get number of properties
        $properties = $jsonObject | Get-Member -MemberType Properties | Select-Object -ExpandProperty Name

        # Check to see if it has properties
        if ($properties.Length -gt 0)
        {
            # Loop through returned properties
            foreach($property in $properties)
            {
                # Make sure we not supposed to ignore it
                if ($null -eq $parentNamespace)
                {
                    # Assign the property value
                    $namespace = $property
                }
                else
                {
                    $namespace = "$parentNamespace.$property"
                }

                # Make sure we're not supposed to ignore it
                if ($settingsToIgnore -notcontains $namespace)
                {
                    # Add the namespace to the array
                    $tempArray += $namespace

                    # Add the returned array from the recursive call
                    $tempArray += Get-AppSettings -jsonObject ($jsonObject.$property) -parentNamespace $namespace -settingsToIgnore $settingsToIgnore
                }
            }
        }
        else
        {
            # Add the value to the array
            if ($null -eq $parentNamespace)
            {
                $namespace = $property
            }
            else
            {
                $namespace = "$parentNamespace.$property"
            }

            $tempArray += $namespace
        }
    }

    return $tempArray
}

# Load the json file
$appSettingsFile = Get-Content -Path (Get-ChildItem -Path $OctopusParameters['Octopus.Action.Package.InstallationDirectoryPath'] | Where-Object {$_.Name -eq "appsettigs.config"}).FullName | ConvertFrom-Json

# Define working variable
$appSettings = @()
$settingsToIgnore = @("octofront.azure")

# Get all the json keys
$appSettings = Get-AppSettings -jsonObject $appSettingsFile -parentNamespace $null -settingsToIgnore $settingsToIgnore

# Get all files that are appsettings.something.json
$otherAppSettingsFiles = Get-ChildItem -Path  $OctopusParameters['Octopus.Action.Package.InstallationDirectoryPath']  | Where-Object {$_.Name -match "^appSettings.*..json"}

# Loop through other files
foreach ($otherFile in $otherAppSettingsFiles)
{
    # Convert the json to PowerShell objects
    $otherSettingsFile = Get-Content -Path $otherFile.FullName  | ConvertFrom-Json

    # Get the json keys
    $otherSettings = Get-AppSettings -jsonObject $otherSettingsFile -parentNamespace $null -settingsToIgnore $settingsToIgnore

    # Compare the properties to see if they are identical
    $results = Compare-Object -ReferenceObject $appSettings -DifferenceObject $otherSettings

    # Check to see if something was returned
    if ($null -ne $results)
    {
        # Extract results
        $resultsError = $results | Out-String

        throw "Differences found in $($otherFile.FullName)`n $resultsError"
    }
} 
```

现在，如果我们忘记向主 appsettings.json 文件添加设置，我们可以检测到它并阻止部署执行！

## 确保应用程序设置有 Octopus 参数

使用上面定义的函数`Get-AppSettings`和`$settingsToIgnore`数组，我们可以使用下面的代码来确保 App Settings 中定义的设置具有相应的 Octopus Deploy 变量:

```
# Load the json file
$appSettingsFile = Get-Content -Path (Get-ChildItem -Path $OctopusParameters['Octopus.Action.Package.InstallationDirectoryPath'] | Where-Object {$_.Name -eq "appsettigs.config"}).FullName | ConvertFrom-Json

# Define working variable
$appSettings = @()
$settingsToIgnore = @("octofront.azure")

# Get all the json keys
$appSettings = Get-AppSettings -jsonObject $appSettingsFile -parentNamespace $null -settingsToIgnore $settingsToIgnore

foreach ($appSetting in $appSettings)
{
    # Check to see if key is present
    if (!$OctopusParameters.ContainsKey($appSetting.Replace(".", ":"))) # Variables have : delimeters for nested json keys
    {
        # Fail the deployment
        throw "Octopus Parameter collection does not contain a value for $($appSetting)"
    }    
} 
```

有了这些步骤，您可以帮助防止部署在。json 配置文件。当然，这个解决方案并没有涵盖所有的场景，但是希望它能让您了解可以采取的预防措施。

愉快的部署！