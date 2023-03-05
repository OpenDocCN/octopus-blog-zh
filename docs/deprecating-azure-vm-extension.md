# 正在删除触手章鱼部署的 Azure VM 扩展

> 原文：<https://octopus.com/blog/deprecating-azure-vm-extension>

触手的 [Azure VM 扩展在 2021 年被弃用，但我们没有将其从市场上移除。它不再与新版本的触手完全兼容，所以我们计划在 2023 年 3 月下旬将其移除。](https://github.com/OctopusDeploy/AzureVMExtension)

在这篇文章中，如果你的工作流程受到影响，我会带你走一遍替代方案。

VM 扩展仅适用于 Azure 中的 Windows VMs。

## 为什么我们要移除触手的 Azure VM 扩展

在最近一篇关于触手的文章中。NET 版本变化，我们解释说我们不再支持旧版本。触须中的 NET 版本。这意味着 Windows 虚拟机没有。NET 4.8 运行时不再运行最新版本的触手。

2022 年，Azure VM 扩展仍然部署了几千个 Windows VMs。几年前，当 Azure 转向 ARM 模板和脚本扩展时，我们就反对这个扩展。因此，当部署到具有较旧操作系统版本的虚拟机时，该扩展不再有效。

我们还将在 2023 年 3 月下旬从市场上移除该扩展，因为它已经超过了折旧期。

## 未来如何调配触手虚拟机

我们建议从现在开始使用 [ARM 模板](https://learn.microsoft.com/en-us/azure/azure-resource-manager/templates/overview)和 [PowerShell DSC 扩展](https://learn.microsoft.com/en-us/azure/virtual-machines/extensions/dsc-windows)来部署触角。

下面是实现这一点的基本步骤。

### 触手版&。网络版

无论您的供应方法是什么，从 6.3 开始的触手都需要。NET 4.8 运行时在 Windows 上运行。我们建议操作系统版本。默认安装. NET 4.8。如果没有，请使用 ARM 模板或 DSC 配置在您的虚拟机上安装。

或者，您可以将触手版本锁定为 6.2，这与大多数旧版本兼容。NET framework 版本。锁定版本后，您可以使用 DSC 配置来安装 6.2 触手并忽略较新的版本。下一节显示了一个 DSC 配置示例。

### 准备 DSC 扩展

Octopus Deploy 提供了一个 [DSC 模块](https://github.com/OctopusDeploy/OctopusDSC)，你可以用它来展开触角。正如在[我们的文档](https://octopus.com/docs/infrastructure/deployment-targets/tentacle/windows/azure-virtual-machines/via-an-arm-template-with-dsc)中解释的，第一步是创建一个包含 DSC 源文件和 DSC 配置的 ZIP 文件。配置文件可以保持简单，或者您可以修改它以接受工作流的参数。比如这里我们增加一个`CommunicationMode`参数，以不同的模式部署触角。

```
configuration OctopusTentacle
{
    param ($ApiKey, $OctopusServerUrl, $Environments, $Roles, $ServerPort, $CommunicationMode)

    Import-DscResource -Module OctopusDSC

    Node "localhost"
    {
        cTentacleAgent OctopusTentacle
        {
            Ensure = "Present"
            State = "Started"

            # Tentacle instance name. Leave it as 'Tentacle' unless you have more
            # than one instance
            Name = "Tentacle"

            # Registration - all parameters required
            ApiKey = $ApiKey
            OctopusServerUrl = $OctopusServerUrl
            Environments = $Environments
            Roles = $Roles

            # How Tentacle will communicate with the server
            CommunicationMode = $CommunicationMode
            ServerPort = $ServerPort

            # Where deployed applications will be installed by Octopus
            DefaultApplicationDirectory = "C:\Applications"

            # Where Octopus should store its working files, logs, packages etc
            TentacleHomeDirectory = "C:\Octopus"
        }
    }
} 
```

如果您将触手锁定到 6.2 版本，请更新 DSC 配置，以便在安装期间只下载 6.2 版本。

```
configuration OctopusTentacle
{
    param ($ApiKey, $OctopusServerUrl, $Environments, $Roles, $ServerPort, $CommunicationMode)

    Import-DscResource -Module OctopusDSC

    Node "localhost"
    {
        cTentacleAgent OctopusTentacle
        {
            Ensure = "Present"
            State = "Started"

            # Tentacle instance name. Leave it as 'Tentacle' unless you have more
            # than one instance
            Name = "Tentacle"

            # Registration - all parameters required
            ApiKey = $ApiKey
            OctopusServerUrl = $OctopusServerUrl
            Environments = $Environments
            Roles = $Roles

            # How Tentacle will communicate with the server
            CommunicationMode = $CommunicationMode
            ServerPort = $ServerPort

            # Where deployed applications will be installed by Octopus
            DefaultApplicationDirectory = "C:\Applications"

            # Where Octopus should store its working files, logs, packages etc
            TentacleHomeDirectory = "C:\Octopus"

            # Lock Tentacle to 6.2 to support older runtimes
            tentacleDownloadUrl = "https://download.octopusdeploy.com/octopus/Octopus.Tentacle.6.2.277.msi"
            tentacleDownloadUrl64 = "https://download.octopusdeploy.com/octopus/Octopus.Tentacle.6.2.277-x64.msi"
        }
    }
} 
```

### 准备手臂模板

#### DSC 扩展的位置

DSC 扩展需要部署到与 VM 相同的位置才能找到它。您可以对 VM 和扩展的位置使用相同的表达式，比如`[resourceGroup().location]`，或者使用一个参数，比如`[parameters('vmLocation')]`。

将位置定义为参数时，该值必须是区域的 ID，例如`australiacentral`或`westus2`。

要查看所有区域及其 id 的列表，使用命令`az account list-locations -o table`。

如果它是一个现有的虚拟机，您也可以通过转到虚拟机页面并查看 **JSON 视图**来找到它。

#### 使用 ARM 模板配置触手虚拟机

最简单的选择是将触手与 VM 和其他相关资源一起部署。一切都是同时定义和部署的，如果需要，可以很容易地重新部署。

下面是 ARM 模板可能的样子。

```
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmAdminUsername": {
      "type": "string",
      "metadata": {
        "description": "Admin username for the Virtual Machine."
      }
    },
    "vmAdminPassword": {
      "type": "securestring",
      "metadata": {
        "description": "Admin password for the Virtual Machine."
      }
    },
    "vmDnsName": {
      "type": "string",
      "metadata": {
        "description": "Unique DNS Name for the Public IP used to access the Virtual Machine."
      }
    },
    "vmSize": {
      "defaultValue": "Standard_D2_v2",
      "type": "string",
      "metadata": {
        "description": "Size of the Virtual Machine"
      }
    },
    "tentacleOctopusServerUrl": {
      "type": "string",
      "metadata": {
        "description": "The URL of the octopus server with which to register"
      }
    },
    "tentacleApiKey": {
      "type": "securestring",
      "metadata": {
        "description": "The Api Key to use to register the Tentacle with the server"
      }
    },
    "tentacleCommunicationMode": {
      "defaultValue": "Listen",
      "allowedValues": [
        "Listen",
        "Poll"
      ],
      "type": "string",
      "metadata": {
        "description": "The type of Tentacle - whether the Tentacle listens for requests from server, or actively polls the server for requests"
      }
    },
    "tentaclePort": {
      "defaultValue": 10933,
      "minValue": 0,
      "maxValue": 65535,
      "type": "int",
      "metadata": {
        "description": "The port on which the Tentacle should listen, when CommunicationMode is set to Listen, or the port on which to poll the server, when CommunicationMode is set to Poll. By default, Tentacle's listen on 10933 and poll the Octopus Server on 10943."
      }
    },
    "tentacleRoles": {
      "type": "string",
      "metadata": {
        "description": "A comma delimited list of roles to apply to the Tentacle"
      }
    },
    "tentacleEnvironments": {
      "type": "string",
      "metadata": {
        "description": "A comma delimited list of environments in which the Tentacle should be placed"
      }
    }
  },
  "variables": {
    "namespace": "octopus",
    "location": "[resourceGroup().location]",
    "tags": {
      "vendor": "Octopus Deploy",
      "description": "Example deployment of Octopus Tentacle to a Windows Server."
    },
    "diagnostics": {
      "storageAccount": {
        "name": "[concat('diagnostics', uniquestring(resourceGroup().id))]"
      }
    },
    "networkSecurityGroupName": "[concat(variables('namespace'), '-nsg')]",
    "publicIPAddressName": "[concat(variables('namespace'), '-publicip')]",
    "vnet": {
      "name": "[concat(variables('namespace'), '-vnet')]",
      "addressPrefix": "10.0.0.0/16",
      "subnet": {
        "name": "[concat(variables('namespace'), '-subnet')]",
        "addressPrefix": "10.0.0.0/24"
      }
    },
    "nic": {
      "name": "[concat(variables('namespace'), '-nic')]",
      "ipConfigName": "[concat(variables('namespace'), '-ipconfig')]"
    },
    "vmName": "[concat(variables('namespace'),'-vm')]"
  },
  "resources": [
    {
      "type": "Microsoft.Storage/storageAccounts",
      "apiVersion": "2021-04-01",
      "name": "[variables('diagnostics').storageAccount.name]",
      "location": "[variables('location')]",
      "tags": {
        "vendor": "[variables('tags').vendor]",
        "description": "[variables('tags').description]"
      },
      "kind": "Storage",
      "sku": {
        "name": "Standard_LRS"
      },
      "properties": {}
    },
    {
      "type": "Microsoft.Network/networkSecurityGroups",
      "apiVersion": "2021-02-01",
      "name": "[variables('networkSecurityGroupName')]",
      "location": "[variables('location')]",
      "tags": {
        "vendor": "[variables('tags').vendor]",
        "description": "[variables('tags').description]"
      },
      "properties": {
        "securityRules": [
          {
            "name": "allow_listening_tentacle",
            "properties": {
              "description": "Allow inbound Tentacle connection",
              "protocol": "Tcp",
              "sourcePortRange": "*",
              "destinationPortRange": "10933",
              "sourceAddressPrefix": "*",
              "destinationAddressPrefix": "*",
              "access": "Allow",
              "priority": 123,
              "direction": "Inbound"
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/publicIPAddresses",
      "name": "[variables('publicIPAddressName')]",
      "apiVersion": "2021-02-01",
      "location": "[variables('location')]",
      "tags": {
        "vendor": "[variables('tags').vendor]",
        "description": "[variables('tags').description]"
      },
      "properties": {
        "publicIPAllocationMethod": "Dynamic",
        "dnsSettings": {
          "domainNameLabel": "[parameters('vmDnsName')]"
        }
      }
    },
    {
      "type": "Microsoft.Network/virtualNetworks",
      "name": "[variables('vnet').name]",
      "apiVersion": "2021-02-01",
      "location": "[variables('location')]",
      "tags": {
        "vendor": "[variables('tags').vendor]",
        "description": "[variables('tags').description]"
      },
      "dependsOn": [
        "[concat('Microsoft.Network/networkSecurityGroups/', variables('networkSecurityGroupName'))]"
      ],
      "properties": {
        "addressSpace": {
          "addressPrefixes": [
            "[variables('vnet').addressPrefix]"
          ]
        },
        "subnets": [
          {
            "name": "[variables('vnet').subnet.name]",
            "properties": {
              "addressPrefix": "[variables('vnet').subnet.addressPrefix]",
              "networkSecurityGroup": {
                "id": "[resourceId('Microsoft.Network/networkSecurityGroups', variables('networkSecurityGroupName'))]"
              }
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Network/networkInterfaces",
      "name": "[variables('nic').name]",
      "apiVersion": "2021-02-01",
      "location": "[variables('location')]",
      "tags": {
        "vendor": "[variables('tags').vendor]",
        "description": "[variables('tags').description]"
      },
      "dependsOn": [
        "[concat('Microsoft.Network/virtualNetworks/', variables('vnet').name)]",
        "[concat('Microsoft.Network/publicIPAddresses/', variables('publicIPAddressName'))]",
        "[concat('Microsoft.Network/networkSecurityGroups/', variables('networkSecurityGroupName'))]"
      ],
      "properties": {
        "ipConfigurations": [
          {
            "name": "[variables('nic').ipConfigName]",
            "properties": {
              "privateIPAllocationMethod": "Dynamic",
              "publicIPAddress": {
                "id": "[resourceId('Microsoft.Network/publicIPAddresses', variables('publicIPAddressName'))]"
              },
              "subnet": {
                "id": "[concat(resourceId('Microsoft.Network/virtualNetworks', variables('vnet').name), '/subnets/', variables('vnet').subnet.name)]"
              }
            }
          }
        ]
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines",
      "name": "[variables('vmName')]",
      "apiVersion": "2021-04-01",
      "location": "[variables('location')]",
      "tags": {
        "vendor": "[variables('tags').vendor]",
        "description": "[variables('tags').description]"
      },
      "dependsOn": [
        "[concat('Microsoft.Storage/storageAccounts/', variables('diagnostics').storageAccount.name)]",
        "[concat('Microsoft.Network/networkInterfaces/', variables('nic').name)]"
      ],
      "properties": {
        "hardwareProfile": {
          "vmSize": "[parameters('vmSize')]"
        },
        "osProfile": {
          "computerName": "[variables('vmName')]",
          "adminUsername": "[parameters('vmAdminUsername')]",
          "adminPassword": "[parameters('vmAdminPassword')]"
        },
        "storageProfile": {
          "imageReference": {
            "publisher": "MicrosoftWindowsServer",
            "offer": "WindowsServer",
            "sku": "2016-Datacenter",
            "version": "latest"
          },
          "osDisk": {
            "createOption": "FromImage",
            "caching": "ReadWrite",
            "managedDisk": {
              "storageAccountType": "Standard_LRS"
            }
          }
        },
        "networkProfile": {
          "networkInterfaces": [
            {
              "id": "[resourceId('Microsoft.Network/networkInterfaces', variables('nic').name)]"
            }
          ]
        },
        "diagnosticsProfile": {
          "bootDiagnostics": {
            "enabled": true,
            "storageUri": "[concat('http://', variables('diagnostics').storageAccount.name, '.blob.core.windows.net')]"
          }
        }
      }
    },
    {
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "name": "[concat(variables('vmName'),'/dscExtension')]",
      "apiVersion": "2021-04-01",
      "location": "[resourceGroup().location]",
      "dependsOn": [
        "[concat('Microsoft.Compute/virtualMachines/', variables('vmName'))]"
      ],
      "properties": {
        "publisher": "Microsoft.Powershell",
        "type": "DSC",
        "typeHandlerVersion": "2.77",
        "autoUpgradeMinorVersion": true,
        "forceUpdateTag": "2",
        "settings": {
          "configuration": {
              "url": "https://url-to-storage/OctopusTentacle.zip",
              "script": "OctopusTentacle.ps1",
              "function": "OctopusTentacle"
          },
          "configurationArguments": {
              "ApiKey": "[parameters('tentacleApiKey')]",
              "OctopusServerUrl": "[parameters('tentacleOctopusServerUrl')]",
              "Environments": "[parameters('tentacleEnvironments')]",
              "Roles": "[parameters('tentacleRoles')]",
              "ServerPort": "[parameters('tentaclePort')]",
              "CommunicationMode":"[parameters('tentacleCommunicationMode')]"
          }
        },
        "protectedSettings": null
      }
    }
  ]
} 
```

在本例中，网络组安全规则允许通过默认端口 10933 与侦听触角对话。轮询触角不需要开放端口，也可以在没有端口的情况下部署。

有关更多示例，请参见我们关于通过 DSC 在 ARM 模板中安装触手[的文档。](https://octopus.com/docs/infrastructure/deployment-targets/tentacle/windows/azure-virtual-machines/via-an-arm-template-with-dsc)

#### 使用 ARM 模板在现有虚拟机上安装触手

还可以使用 ARM 模板将触手部署到现有的虚拟机上。需要将扩展部署到与 VM 相同的区域才能找到它。您必须提供虚拟机的名称和位置。

下面是 ARM 模板可能的样子。

```
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "vmName": {
      "type": "string",
      "metadata": {
        "description": "Name of the Virtual Machine to run the extension on"
      }
    },
    "vmLocation": {
        "type": "string",
      "metadata": {
        "description": "Region id of the Virtual Machine, e.g. westus2"
      }
    },
    "tentacleOctopusServerUrl": {
      "type": "string",
      "metadata": {
        "description": "The URL of the octopus server with which to register"
      }
    },
    "tentacleApiKey": {
      "type": "securestring",
      "metadata": {
        "description": "The Api Key to use to register the Tentacle with the server"
      }
    },
    "tentacleCommunicationMode": {
      "defaultValue": "Listen",
      "allowedValues": ["Listen", "Poll"],
      "type": "string",
      "metadata": {
        "description": "The type of Tentacle - whether the Tentacle listens for requests from server, or actively polls the server for requests"
      }
    },
    "tentaclePort": {
      "defaultValue": 10933,
      "minValue": 0,
      "maxValue": 65535,
      "type": "int",
      "metadata": {
        "description": "The port on which the Tentacle should listen, when CommunicationMode is set to Listen, or the port on which to poll the server, when CommunicationMode is set to Poll. By default, Tentacle's listen on 10933 and poll the Octopus Server on 10943."
      }
    },
    "tentacleRoles": {
      "type": "string",
      "metadata": {
        "description": "A comma delimited list of roles to apply to the Tentacle"
      }
    },
    "tentacleEnvironments": {
      "type": "string",
      "metadata": {
        "description": "A comma delimited list of environments in which the Tentacle should be placed"
      }
    }
  },
  "resources": [
    {
      "type": "Microsoft.Compute/virtualMachines/extensions",
      "name": "[concat(parameters('vmName'),'/dscTentacleExtension')]",
      "apiVersion": "2021-04-01",
      "location": "[parameters('vmLocation')]",
      "properties": {
        "publisher": "Microsoft.Powershell",
        "type": "DSC",
        "typeHandlerVersion": "2.77",
        "autoUpgradeMinorVersion": true,
        "forceUpdateTag": "2",
        "settings": {
          "configuration": {
            "url": "https://url-to-storage/OctopusTentacle.zip",
            "script": "OctopusTentacle.ps1",
            "function": "OctopusTentacle"
          },
          "configurationArguments": {
            "ApiKey": "[parameters('tentacleApiKey')]",
            "OctopusServerUrl": "[parameters('tentacleOctopusServerUrl')]",
            "Environments": "[parameters('tentacleEnvironments')]",
            "Roles": "[parameters('tentacleRoles')]",
            "ServerPort": "[parameters('tentaclePort')]",
            "CommunicationMode":"[parameters('tentacleCommunicationMode')]"
          }
        },
        "protectedSettings": null
      }
    }
  ]
} 
```

### 部署 ARM 模板

您可以从以下位置部署 ARM 模板:

## 后续步骤

1.  我们计划在 2023 年 3 月 8 日星期三将触手 6.3 重新发布到我们的下载页面和 Chocolatey。由于这篇文章中描述的 Azure VM 扩展问题，我们不得不从这些来源中获取最新的信息。如果您对扩展有问题，这可能是原因。
2.  我们计划在 2023 年 3 月底从市场上移除 Azure VM 扩展，以完成弃用过程。

## 结论

随着 Azure VM 扩展的退役，我们现在建议您使用 ARM 模板和 DSC 扩展在 Azure 中部署 Windows 触手 VM。这使您能够更好地控制如何部署虚拟机和触角。它还让我们更有信心地更新触手，因此我们可以为您带来更强大、更流畅的部署体验。

愉快的部署！