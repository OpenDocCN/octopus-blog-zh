# 跨空间共享工作人员- Octopus 部署

> 原文：<https://octopus.com/blog/sharing-workers-spaces>

Octopus Deploy 中的空间是硬墙，不允许你在它们之间共享任何东西(用户除外)。这对于隔离来说很好，但对于你想在所有空间共享的东西来说可能会有问题，比如工人。共享工人是我们为支持我们的[示例](https://samples.octopus.app)实例而解决的一个场景。

在这篇文章中，我解释了我们的解决方案，它可以自动共享多个空间的工作人员。

## 问题是

我们的 [Samples](https://samples.octopus.app) 实例的基础设施每天都在创建和销毁。随着示例实例的增长，对分配云实例(一个 Windows 和一个 Linux)的[动态工作器](https://octopus.com/docs/infrastructure/workers/dynamic-worker-pools)提出了更多的要求。

我们遇到了资源争用问题，因为我们用来创建基础设施的[操作手册](https://octopus.com/docs/runbooks)经常会使用 **[运行 Octopus 部署操作手册](https://library.octopus.com/step-templates/0444b0b3-088e-4689-b755-112d1360ffe3/actiontemplate-run-octopus-deploy-runbook)** 步骤模板。您可以将此步骤模板配置为等待调用的运行手册完成；然而，这有时意味着父进程将等待子进程完成，但子进程需要独占访问(参见我们的[帖子解释 Workers](https://octopus.com/blog/workers-explained) 了解可能发生这种情况的条件)，并将等待第一个任务完成。

## 解决方案

我们需要更多的工人来完成这些任务。我们提出了两种解决方案:

*   敬业的工人
*   共享工人

### 解决方案 1:敬业的员工

我们解决这个问题的第一个迭代是让每个空间创建自己的专用工人。虽然这样做很好，但有两个问题:

*   工人们一天中 99%的时间都无所事事
*   随着空间的增加，我们的成本也增加了

我们需要重新评估我们的云资源使用情况，并寻求成本节约。我们将工作人员以及数据库服务器实例之类的东西确定为可以共享而不是专用的项目。

### 解决方案 2:共享工人

样本中的每个空间都创建了使用符合其需求的云提供商的工作人员。为了支持这一点，我们在所有 3 个云提供商中创建了工作人员，与所有空间共享。

通过 Terraform，我们使用云提供商的扩展功能创建了工作人员，因此，如果我们需要更多或更少的工作人员，我们只需相应地调整规模。配置工作人员使用包含以下步骤的操作手册:

*   获取空间列表
*   创建工人
*   等待工人自己注册
*   向剩余空间添加工作人员

#### 获取空间列表

为了向所有空间添加工人，我们首先需要收集实例上所有空间的列表。**获取空间列表**步骤检索列表并设置以下输出变量:

*   `InitialSpaceName` -我们的空间列表中第一个空间的名称，该值用于虚拟机上运行的脚本，以便它们可以将自己注册到实例。
*   `InitialSpaceId` -工作人员将被添加到的初始空间的 ID。
*   `RemainingSpaceIds` -以逗号分隔的剩余空间 id 列表，用于添加工人。
*   `WorkerPoolName` -要在空间中注册和添加的工人池的名称。Samples 使用 Terraform 在所有空间创建了一个一致的工人池列表。

```
function Get-OctopusItems
{
    # Define parameters
    param(
        $OctopusUri,
        $ApiKey,
        $SkipCount = 0
    )

    # Define working variables
    $items = @()
    $skipQueryString = ""
    $headers = @{"X-Octopus-ApiKey"="$ApiKey"}

    # Check to see if there there is already a querystring
    if ($octopusUri.Contains("?"))
    {
        $skipQueryString = "&skip="
    }
    else
    {
        $skipQueryString = "?skip="
    }

    $skipQueryString += $SkipCount

    # Get intial set
    $resultSet = Invoke-RestMethod -Uri "$($OctopusUri)$skipQueryString" -Method GET -Headers $headers

    # Check to see if it returned an item collection
    if ($resultSet.Items)
    {
        # Store call results
        $items += $resultSet.Items

        # Check to see if resultset is bigger than page amount
        if (($resultSet.Items.Count -gt 0) -and ($resultSet.Items.Count -eq $resultSet.ItemsPerPage))
        {
            # Increment skip count
            $SkipCount += $resultSet.ItemsPerPage

            # Recurse
            $items += Get-OctopusItems -OctopusUri $OctopusUri -ApiKey $ApiKey -SkipCount $SkipCount
        }
    }
    else
    {
        return $resultSet
    }

    # Return results
    return $items
}

# Define variables
$baseUrl = $OctopusParameters['Samples.Octopus.Url'] 
$apiKey = $OctopusParameters['Samples.Octopus.Api.Key']
$header = @{ "X-Octopus-ApiKey" = $apiKey }
$initialSpaceName = ""
$initialSpaceId = ""
$remainingSpaceIds = @()
$workerPoolName = "$($OctopusParameters['Project.CloudProvider.Folder.Name']) Worker Pool TF"

# Get all spaces
Write-Host "Getting list of all spaces on $baseUrl ..."
$spaces = Get-OctopusItems -OctopusUri "$baseUrl/api/spaces" -ApiKey $apiKey

# Loop through the spaces
foreach ($space in $spaces)
{
    # Get worker pools of space
    Write-Host "Getting all worker pools for space $($space.Name) ..."
    $workerPools = Get-OctopusItems -OctopusUri "$baseUrl/api/$($space.Id)/workerPools" -ApiKey $apiKey

    # Check to see if it has the pool we're looking for
    if ($workerPools.Name -contains $workerPoolName)
    {
        # Check to see if we have an initial space
        if ([string]::IsNullOrWhitespace($initialSpaceName))
        {
            # Assign the values
            Write-Host "Initial Space: $($space.name)"
            $initialSpaceName = $space.Name
            $initialSpaceId = $space.Id
        }
        else
        {
            # Add to list
            Write-Host "Adding space: $($space.Name)($($space.Id))"
            $remainingSpaceIds += $space.Id
        }
    }
}

# Set output variables
Write-Host "Setting output variable InitialSpaceName to $initialSpaceName"
Set-OctopusVariable -name "InitialSpaceName" -value $initialSpaceName
Write-Host "Setting output variable InitialSpaceId to $initialSpaceId"
Set-OctopusVariable -name "InitialSpaceId" -value $initialSpaceId
Set-OctopusVariable -name "RemainingSpaceIds" -value ($remainingSpaceIds -join ",")
Write-Host "Setting output variable WorkerPoolName to $workerPoolName"
Set-OctopusVariable -name "WorkerPoolName" -value $workerPoolName 
```

#### 创建工人

对于我们的共享工作人员，我们将 Terraform 文件分离到云提供商指定的文件夹中。下面是我们为每个提供者创建的 Terraform 的片段(参见 GitHub 中的完整实现[):](https://github.com/OctopusSamples/IaC/tree/master/octopus-samples-instances/shared-workers-terraform)

<details><summary>AWS 工人地形</summary></details>

```
resource "aws_iam_instance_profile" "linux-worker-profile" {
    name = var.octopus_aws_instance_profile_name
    role = var.octopus_aws_role_name
}

resource "aws_launch_configuration" "linux-worker-launchconfig" {
    name_prefix = var.octopus_aws_launch_configuration_name
    image_id = "${var.octopus_aws_linux_ami_id}"
    instance_type = var.octopus_aws_ec2_instance_type

    iam_instance_profile = "${aws_iam_instance_profile.linux-worker-profile.name}"

    security_groups = ["${var.octopus_aws_security_group_id}"]

    # script to run when created
    user_data = "${file("../configure-tentacle.sh")}"

    # root disk
    root_block_device {
        volume_size           = "30"
        delete_on_termination = true
    }
}

resource "aws_autoscaling_group" "linux-worker-autoscaling" {
    name = var.auto_scaling_group_name
    vpc_zone_identifier = var.octopus_aws_subnets
    launch_configuration = "${aws_launch_configuration.linux-worker-launchconfig.name}"
    min_size = var.octopus_aws_autoscalinggroup_size
    max_size = var.octopus_aws_autoscalinggroup_size
    health_check_grace_period = 300
    health_check_type = "EC2"
    force_delete = true

    tag {
        key = "Name"
        value = "Samples Linux Worker"
        propagate_at_launch = true
    }
} 
```

<details><summary>天蓝色工人地形</summary></details>

```
// Define resource group
resource "azurerm_resource_group" "octopus-samples-azure-workers" {
  name      = var.octopus_azure_resourcegroup_name
  location  = var.octopus_azure_location
  tags = var.tags
}

// Define virtual network
resource "azurerm_virtual_network" "octopus-samples-workers-virtual-network" {
  name                = "octopus-samples-workers"
  address_space       = ["10.0.0.0/16"]
  location            = var.octopus_azure_location
  resource_group_name = var.octopus_azure_resourcegroup_name
  depends_on = [
     azurerm_resource_group.octopus-samples-azure-workers
  ]
  tags = var.tags
}

// Define subnet
resource "azurerm_subnet" "octopus-samples-workers-subnet" {
  name                 = "octopus-samples-workers-subnet"
  resource_group_name  = var.octopus_azure_resourcegroup_name
  virtual_network_name = azurerm_virtual_network.octopus-samples-workers-virtual-network.name
  address_prefixes     = ["10.0.2.0/24"]
  depends_on = [
     azurerm_resource_group.octopus-samples-azure-workers,
     azurerm_virtual_network.octopus-samples-workers-virtual-network
  ]  
}

// Define azure scale set
resource "azurerm_linux_virtual_machine_scale_set" "samples-azure-workers" {
  name                = var.octopus_azure_scaleset_name
  resource_group_name = var.octopus_azure_resourcegroup_name
  location            = var.octopus_azure_location
  sku                 = var.octopus_azure_vm_size
  instances           = var.octopus_azure_vm_instance_count
  admin_username      = var.octopus_azure_vm_admin_username
  admin_password =  var.octopus_azure_vm_admin_password
  disable_password_authentication = false
  user_data = "${base64encode(file("../configure-tentacle.sh"))}"

  identity {
    type = "SystemAssigned"
  }

  source_image_reference {
    publisher = "Canonical"
    offer     = "UbuntuServer"
    sku       = var.octopus_azure_vm_sku
    version   = "latest"
  }

  os_disk {
    storage_account_type = "Standard_LRS"
    caching              = "ReadWrite"
  }

  network_interface {
    name    = "example"
    primary = true

    ip_configuration {
      name      = "internal"
      primary   = true
      subnet_id = azurerm_subnet.octopus-samples-workers-subnet.id
    }
  }
  tags = var.tags
} 
```

<details><summary>GCP 工人地形</summary></details>

```
resource "google_compute_instance" "vm_instance" {
  count        = var.instance_count
  name         = "${var.instance_name}-${count.index + 1}"
  machine_type = var.instance_size

  boot_disk {
    initialize_params {
      image = var.instance_osimage
      size = 30
    }
  }

  network_interface {
    network = "default" #google_compute_network.vpc_network.name

    access_config {
      // Ephemeral public IP - needed to send and receive traffic directly to and from outside network
    }
  }

  metadata_startup_script = file("../configure-tentacle.sh")

  service_account {
    email = google_service_account.database_service_account.email
    scopes = ["cloud-platform"]
  }

  tags = ["octopus-samples-worker"]
}

output "ip" {
  value = google_compute_instance.vm_instance[*].network_interface[0].access_config[0].nat_ip
} 
```

Terraform 包包含一个 Bash 脚本，运行在由云扩展技术创建的虚拟机上。当被执行时，VM 从**获取空间列表**步骤的输出变量中向空间和池注册自己。

```
#!/bin/bash
serverUrl="#{Samples.Octopus.Url}"
serverCommsPort="10943"
apiKey="#{Samples.Octopus.Api.Key}"
name=$HOSTNAME
configFilePath="/etc/octopus/default/tentacle-default.config"
applicationPath="/home/Octopus/Applications/"
workerPool="#{Octopus.Action[Get Samples Spaces].Output.WorkerPoolName}"
machinePolicy="Default Machine Policy"
space="#{Octopus.Action[Get Samples Spaces].Output.InitialSpaceName}"

# Install Tentacle
sudo apt-key adv --fetch-keys "https://apt.octopus.com/public.key"
sudo add-apt-repository "deb https://apt.octopus.com/ focal main"
sudo apt-get update
sudo apt-get install tentacle -y

# Install Docker
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo apt-key add -
sudo add-apt-repository "deb [arch=amd64] https://download.docker.com/linux/ubuntu focal stable"
sudo apt-get install docker-ce docker-ce-cli containerd.io -y

# Install wget
sudo apt-get install wget -y

# Download the Microsoft repository GPG keys
wget https://packages.microsoft.com/config/debian/10/packages-microsoft-prod.deb

# Register the Microsoft repository GPG keys
sudo dpkg -i packages-microsoft-prod.deb

# Update the list of products
sudo apt-get update

# Install PowerShell
sudo apt-get install -y powershell

# Pull worker tools image
sudo docker pull #{Project.Docker.WorkerToolImage}:#{Project.Docker.WorkerToolImageTag}

# Configure and register worker
sudo /opt/octopus/tentacle/Tentacle create-instance --config "$configFilePath" --instance "$name"
sudo /opt/octopus/tentacle/Tentacle new-certificate --if-blank
sudo /opt/octopus/tentacle/Tentacle configure --noListen True --reset-trust --app "$applicationPath"
echo "Registering the worker $name with server $serverUrl"
sudo /opt/octopus/tentacle/Tentacle service --install --start
sudo /opt/octopus/tentacle/Tentacle register-worker --server "$serverUrl" --apiKey "$apiKey" --name "$name"  --comms-style "TentacleActive" --server-comms-port $serverCommsPort --workerPool "$workerPool" --policy "$machinePolicy" --space "$space"
sudo /opt/octopus/tentacle/Tentacle service --restart 
```

#### 等待工人自己注册

在我们可以将工人添加到其他空间之前，工人需要注册他们自己。这一步监视工作线程池，直到注册了所需数量的工作线程。

```
# Define parameters 
$baseUrl = $OctopusParameters['Samples.Octopus.Url'] 
$apiKey = $OctopusParameters['Samples.Octopus.Api.Key']
$header = @{ "X-Octopus-ApiKey" = $apiKey }
$spaceId = $OctopusParameters['Octopus.Action[Get Samples Spaces].Output.InitialSpaceId']
$spaceName = $OctopusParameters['Octopus.Action[Get Samples Spaces].Output.InitialSpaceName']
$workerPoolName = $OctopusParameters['Octopus.Action[Get Samples Spaces].Output.WorkerPoolName']

if ($baseUrl.EndsWith("/"))
{
    $baseUrl = $baseUrl.SubString(0, $baseUrl.LastIndexOf("/"))
}

# Get worker pool
Write-Host "Getting reference to $workerPoolName in space $spaceName ..."
$workerPool = ((Invoke-RestMethod -Method Get -Uri "$baseUrl/api/$($spaceId)/workerpools/all" -Headers @{"X-Octopus-ApiKey"="$apiKey"}) | Where-Object {$_.Name -eq $workerPoolName})

# Check worker pool
if ($null -ne $workerPool)
{
  # Get all workers
  Write-Host "Checking to see if workers have registered themselves ..."
  $workers = (Invoke-RestMethod -Method Get -Uri "$baseUrl/api/$($spaceId)/workerpools/$($workerPool.Id)/workers" -Headers @{"X-Octopus-ApiKey"="$apiKey"})

  $retries = 20

  $cloudProvider = $OctopusParameters['Project.CloudProvider.Folder.Name']
  $numberOfWorkersKey = $OctopusParameters.Keys | Where-Object {$_ -like "*$cloudProvider*" -and $_ -like "*Instance.Count*"}
  $numberOfWorkers = $OctopusParameters[$numberOfWorkersKey]

  while ($workers.Items.Count -ne $numberOfWorkers)
  {
    if ($retries -gt 0)
    {
        Write-Host "Waiting 60 seconds for $numberOfWorkers workers to register themselves, $retries tries remaining ..."
        Start-Sleep -Seconds 60
        $retries--
        $workers = (Invoke-RestMethod -Method Get -Uri "$baseUrl/api/$($spaceId)/workerpools/$($workerPool.Id)/workers" -Headers @{"X-Octopus-ApiKey"="$apiKey"})
    }
    else
    {
        Write-Error "Workers didn't show up in time!"
    }
  }
} 
```

#### 向剩余空间添加工作人员

在我们将工人添加到第一个空间之后，我们使用 API 将他们添加到剩余的空间。

```
function Get-OctopusItems
{
    # Define parameters
    param(
        $OctopusUri,
        $ApiKey,
        $SkipCount = 0
    )

    # Define working variables
    $items = @()
    $skipQueryString = ""
    $headers = @{"X-Octopus-ApiKey"="$ApiKey"}

    # Check to see if there there is already a querystring
    if ($octopusUri.Contains("?"))
    {
        $skipQueryString = "&skip="
    }
    else
    {
        $skipQueryString = "?skip="
    }

    $skipQueryString += $SkipCount

    # Get intial set
    $resultSet = Invoke-RestMethod -Uri "$($OctopusUri)$skipQueryString" -Method GET -Headers $headers

    # Check to see if it returned an item collection
    if ($resultSet.Items)
    {
        # Store call results
        $items += $resultSet.Items

        # Check to see if resultset is bigger than page amount
        if (($resultSet.Items.Count -gt 0) -and ($resultSet.Items.Count -eq $resultSet.ItemsPerPage))
        {
            # Increment skip count
            $SkipCount += $resultSet.ItemsPerPage

            # Recurse
            $items += Get-OctopusItems -OctopusUri $OctopusUri -ApiKey $ApiKey -SkipCount $SkipCount
        }
    }
    else
    {
        return $resultSet
    }

    # Return results
    return $items
}

# Define variables
$baseUrl = $OctopusParameters['Samples.Octopus.Url'] 
$apiKey = $OctopusParameters['Samples.Octopus.Api.Key']
$header = @{ "X-Octopus-ApiKey" = $apiKey }
$remainingSpaceIds = $OctopusParameters['Octopus.Action[Get Samples Spaces].Output.RemainingSpaceIds'].Split(",")
$initialSpaceId = $OctopusParameters['Octopus.Action[Get Samples Spaces].Output.InitialSpaceId']
$initialWorkerPoolName = "$($OctopusParameters['Project.CloudProvider.Folder.Name']) Worker Pool TF"

# Get registered workers
$initialWorkerPool = (Get-OctopusItems -OctopusUri "$baseUrl/api/$initialSpaceId/workerPools" -ApiKey $apiKey | Where-Object {$_.Name -eq $initialWorkerPoolName})
$workers = Get-OctopusItems -OctopusUri "$baseUrl/api/$initialSpaceId/workerPools/$($initialWorkerPool.Id)/workers" -ApiKey $apiKey

# Loop through the spaces
foreach ($spaceId in $remainingSpaceIds)
{
    $workerPool = (Get-OctopusItems -OctopusUri "$baseUrl/api/$spaceId/workerPools" -ApiKey $apiKey | Where-Object {$_.Name -eq $initialWorkerPoolName})    

    # Check worker pool
    Write-Host "Verifying that space Id $spaceId has a worker pool called $initialWorkerPoolName ..."
    if ($null -ne $workerPool)
    {
        # Get default machine policy
        $machinePolicy = (Get-OctopusItems -OctopusUri "$baseUrl/api/$spaceId/machinepolicies" -ApiKey $apiKey | Where-Object {$_.Name -eq "Default Machine Policy"})    

        # Loop through workers
        foreach ($worker in $workers)
        {

            # Build JSON payload
            $jsonPayload = @{
                Name = $worker.Name
                MachinePolicyId = $machinePolicy.Id
                IsDisabled = $worker.IsDisabled
                HealthStatus = $worker.HealthStatus
                HasLatestCalamari = $worker.HasLatestCalamari
                IsInProcess = $true
                EndPoint = $worker.Endpoint
                WorkerPoolIds = @($workerPool.Id)
            }

            try
            {
                # Add worker
                Write-Host "Adding $($worker.Name) to space Id $spaceId..."

                Invoke-RestMethod -Method Post -Uri "$baseUrl/api/$($spaceId)/workers" -Body ($jsonPayload | ConvertTo-Json -Depth 10) -Headers $header
            }
            catch
            {
                Write-Highlight "An error occured adding $($worker.Name) to space Id $spaceId"
                Write-Warning "An error occured adding $($worker.Name) to space Id $spaceId"
                Write-Host "StatusCode:" $_.Exception.Response.StatusCode.value__ 
                Write-Host "StatusDescription:" $_.Exception.Response.StatusDescription                   
            }
        }
    }
} 
```

## 结论

在我们的 [Samples](https://samples.octopus.app) 实例的所有空间中共享工作人员让我们更有效地整合和使用资源。我希望这篇文章能给你一些如何做同样事情的想法。

愉快的部署！