# 为复杂安装使用 Azure 自定义脚本扩展- Octopus Deploy

> 原文：<https://octopus.com/blog/azure-script-extension>

[![Azure custom script extensions](img/ab2c188e4478636dcf60758215a399d5.png)](#)

首次启动虚拟机(VM)时，运行自定义脚本通常很有用。该脚本可能会安装附加软件、配置虚拟机或执行一些其他管理任务。

在 Azure 中，[自定义脚本扩展](https://docs.microsoft.com/en-au/azure/virtual-machines/extensions/custom-script-windows)提供了运行脚本的能力。当 Windows Azure 虚拟机与 Chocolatey 等工具相结合时，几乎可以用任何您需要的软件来初始化新的虚拟机。

然而，你需要考虑 Windows 的一些边缘情况，在这篇博客文章中，我们将深入研究通过 Azure 自定义脚本扩展执行复杂安装的细节。

## 一个简单的例子——配置一个 Windows Azure 虚拟机

让我们从一个非常简单的例子开始。以下 Terraform 示例脚本使用自定义脚本扩展配置 Windows 虚拟机:

```
variable "resgroupname" {
  type = "string"
}

resource "azurerm_resource_group" "test" {
  name     = "${var.resgroupname}"
  location = "Australia East"
}

resource "azurerm_public_ip" "test" {
  name                    = "test-pip"
  location                = "${azurerm_resource_group.test.location}"
  resource_group_name     = "${azurerm_resource_group.test.name}"
  allocation_method       = "Dynamic"
  idle_timeout_in_minutes = 30
}

resource "azurerm_virtual_network" "test" {
  name                = "acctvn"
  address_space       = ["10.0.0.0/16"]
  location            = "${azurerm_resource_group.test.location}"
  resource_group_name = "${azurerm_resource_group.test.name}"
}

resource "azurerm_subnet" "test" {
  name                 = "acctsub"
  resource_group_name  = "${azurerm_resource_group.test.name}"
  virtual_network_name = "${azurerm_virtual_network.test.name}"
  address_prefix       = "10.0.2.0/24"
}

resource "azurerm_network_interface" "test" {
  name                = "acctni"
  location            = "${azurerm_resource_group.test.location}"
  resource_group_name = "${azurerm_resource_group.test.name}"

  ip_configuration {
    name                          = "testconfiguration1"
    subnet_id                     = "${azurerm_subnet.test.id}"
    private_ip_address_allocation = "Dynamic"
    public_ip_address_id          = "${azurerm_public_ip.test.id}"
  }
}

resource "azurerm_storage_account" "test" {
  name                     = "${var.resgroupname}"
  resource_group_name      = "${azurerm_resource_group.test.name}"
  location                 = "${azurerm_resource_group.test.location}"
  account_tier             = "Standard"
  account_replication_type = "LRS"
}

resource "azurerm_storage_container" "test" {
  name                  = "vhds"
  resource_group_name   = "${azurerm_resource_group.test.name}"
  storage_account_name  = "${azurerm_storage_account.test.name}"
  container_access_type = "private"
}

resource "azurerm_virtual_machine" "test" {
  name                  = "acctvm"
  location              = "${azurerm_resource_group.test.location}"
  resource_group_name   = "${azurerm_resource_group.test.name}"
  network_interface_ids = ["${azurerm_network_interface.test.id}"]
  vm_size               = "Standard_D2_v3"

  storage_image_reference {
    publisher = "MicrosoftWindowsServer"
    offer     = "WindowsServer"
    sku       = "2019-Datacenter"
    version   = "latest"
  }

  storage_os_disk {
    name          = "osdisk"
    vhd_uri       = "${azurerm_storage_account.test.primary_blob_endpoint}${azurerm_storage_container.test.name}/osdisk.vhd"
    caching       = "ReadWrite"
    create_option = "FromImage"
  }

  os_profile {
    computer_name  = "hostname"
    admin_username = "testadmin"
    admin_password = "passwordoeshere"
  }

  os_profile_windows_config {
    enable_automatic_upgrades = false
    provision_vm_agent = true
  }
}

resource "azurerm_virtual_machine_extension" "test" {
  name                 = "hostname"
  location             = "${azurerm_resource_group.test.location}"
  resource_group_name  = "${azurerm_resource_group.test.name}"
  virtual_machine_name = "${azurerm_virtual_machine.test.name}"
  publisher            = "Microsoft.Compute"
  type                 = "CustomScriptExtension"
  type_handler_version = "1.9"

  protected_settings = <<PROTECTED_SETTINGS
    {
      "commandToExecute": "powershell.exe -Command \"./chocolatey.ps1; exit 0;\""
    }
  PROTECTED_SETTINGS

  settings = <<SETTINGS
    {
        "fileUris": [
          "https://gist.githubusercontent.com/mcasperson/c815ac880df481418ff2e199ea1d0a46/raw/5d4fc583b28ecb27807d8ba90ec5f636387b00a3/chocolatey.ps1"
        ]
    }
  SETTINGS
} 
```

这个脚本的重要部分是`azurerm_virtual_machine_extension`资源。在`settings`字段中，我们有一个 JSON blob 列表脚本要下载到`fileUris`数组中，在`protected_settings`字段中，我们有另一个 JSON blob，它带有一个`commandToExecute`字符串，定义了我们将要运行的脚本的入口点。

在本例中，我们从 GitHub Gist 下载一个 PS1 PowerShell 脚本文件，GitHub Gist 安装 Chocolatey，然后安装 Notepad++和 Chocolatey 客户端:

```
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
choco install notepadplusplus -y 
```

下载的脚本文件然后由分配给`commandToExecute`字段的字符串运行，带有一个`exit 0`以确保脚本扩展注册我们的脚本成功运行:

```
 "commandToExecute": "powershell.exe -Command \"./chocolatey.ps1; exit 0;\"" 
```

试图将 PowerShell 脚本编码到`commandToExecute` JSON 字符串中很快变得难以管理。使用`fileUris`下载脚本是一个更好的解决方案，如果需要，脚本可以托管在 Azure blob 存储中以获得更好的安全性。

这个例子非常简单，对于简单的软件安装，这就是我们所需要的。不幸的是，并不是所有的软件都这么容易安装。

## 在 Azure 虚拟机上自动安装 SQL Server

为了查看此示例的失败之处，我们将尝试安装 Microsoft SQL Server Express:

```
Set-ExecutionPolicy Bypass -Scope Process -Force
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))
choco install sql-server-express -y 
```

SQL Server 显然是一个比 Notepad++更复杂的软件，在试图安装它时，我们遇到了一些错误。在 Chocolatey 日志中，我们可以看到 SQL Server 安装失败:

```
2019-11-06 05:47:59,622 2240 [WARN ] - WARNING: May not be able to find 'C:\windows\TEMP\chocolatey\sql-server-express\2017.20190916\SQLEXPR\setup.exe'. Please use full path for executables.
2019-11-06 05:47:59,751 2240 [ERROR] - ERROR: Exception calling "Start" with "0" argument(s): "The system cannot find the file specified" 
```

出现此错误是因为运行自定义脚本的系统帐户[不能与 SQL Server 安装一起使用。我们需要一种方式来运行安装作为一个普通的管理员帐户。](https://www.powershellmagazine.com/2014/04/30/understanding-azure-custom-script-extension/)

## 在新帐户下运行 PowerShell 脚本

PowerShell 为以不同用户身份运行代码提供了一个方便的解决方案。通过调用`Invoke-Command`，我们可以作为我们选择的用户在本地(或者远程，如果需要的话)VM 上执行一个脚本块。

下面的代码显示了我们如何构建一个凭证对象，并将其传递给`Invoke-Command`命令，以管理员用户的身份执行 Chocolaty 安装:

用户名必须采用`machinename\username`格式，该命令才能正常工作。

```
$securePassword = ConvertTo-SecureString 'passwordgoeshere' -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential 'hostname\testadmin', $securePassword
Invoke-Command -ScriptBlock {choco install sql-server-express -y} -ComputerName hostname -Credential $credential 
```

这个*几乎和*一样有效，但是现在我们得到了这个错误:

```
Inner exception type: System.InvalidOperationException
       Message:
               There was an error generating the XML document.
       HResult : 0x80131509 
```

不幸的是，对`Invoke-Command`的默认调用没有授予足够的权限来完成安装。我们遇到了双跳问题，这导致了上面显示的异常。

## 支持 PowerShell 双跃点

在远程机器上执行代码，或者在同一台机器上执行“远程”代码，被认为是第一跳。让代码继续访问另一台机器被认为是一个 [PowerShell 双跳](https://docs.microsoft.com/en-us/powershell/scripting/learn/remoting/ps-remoting-second-hop?view=powershell-6)，当使用`Invoke-Command`时，这个第二跳被默认阻止。我们的 SQL Server 安装遇到的其他调用也被视为第二跳。

为了允许 SQL Server 安装完成，我们需要允许这种双跃点发生。我们通过利用 CredSSP 身份验证来做到这一点。

第一步是让我们的机器承担`Server`角色，这意味着它可以接受来自客户端的凭证:

```
Enable-WSManCredSSP -Role Server -Force 
```

然后我们需要让我们的机器承担`Client`角色，这意味着它可以向服务器发送凭证。我们同时启用客户机和服务器角色，因为我们使用`Invoke-Command`在同一台机器上运行命令:

```
Enable-WSManCredSSP -Role Client -DelegateComputer * -Force 
```

因为我们运行代码的帐户是本地帐户(与域帐户相反)，所以我们需要允许使用 NTLM 帐户。这是通过设置注册表项来实现的:

```
New-Item -Path HKLM:\SOFTWARE\Policies\Microsoft\Windows\CredentialsDelegation -Name AllowFreshCredentialsWhenNTLMOnly -Force
New-ItemProperty -Path HKLM:\SOFTWARE\Policies\Microsoft\Windows\CredentialsDelegation\AllowFreshCredentialsWhenNTLMOnly -Name 1 -Value * -PropertyType String 
```

有了这些更改，我们就可以使用 CredSSP 身份验证运行 Chocolaty 安装了。这将启用双跃点，我们的 SQL Server 安装将成功完成:

```
$securePassword = ConvertTo-SecureString 'passwordgoeshere' -AsPlainText -Force
$credential = New-Object System.Management.Automation.PSCredential 'hostname\testadmin', $securePassword
Invoke-Command -Authentication CredSSP -ScriptBlock {choco install sql-server-express -y} -ComputerName hostname -Credential $credential 
```

完整的脚本已经保存在这个[要点](https://gist.githubusercontent.com/mcasperson/6b87b519bd0ab1d093e697b33938ed3b/raw/b635114f552ea230e08828c63274316323799386/chocolatey.ps1)中。

## 结论

虽然使用 Azure 自定义脚本扩展，大多数脚本都将按预期运行，但有时您需要以管理员用户的身份运行脚本。在本文中，我们看到了如何利用 CredSSP 身份验证来确保脚本以安装 SQL Server 等复杂应用程序所需的最高权限运行。