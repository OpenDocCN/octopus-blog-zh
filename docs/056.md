# 使用 Octopus Runbooks 自动化支持电子邮件- Octopus Deploy

> 原文：<https://octopus.com/blog/automating-support-emails-runbooks>

你可以用 Octopus Runbooks 自动完成许多有用的任务，特别是关于基础设施的管理。一个微妙但同样有用的事情是章鱼可以让人们知道什么时候有问题。

根据您需要的信息和对您的支持团队有用的信息，有几种方法可以实现这一点。

让我们看两个例子。我们用两个 run book 创建了一个[样本 Octopus 实例，所以您可以很容易地理解。](https://tenpillars.octopus.app/app#/Spaces-103)

## 在 Octopus 中设置 SMTP 连接

您必须在 Octopus 中输入 SMTP 设置，runbook steps 才能向您的支持团队发送电子邮件。如果您不知道您的 SMTP 服务器或端口的详细信息，请咨询您的运营团队。

1.  点击顶部菜单中的**配置**。
2.  从左侧菜单中点击 **SMTP** 。
3.  填写以下字段，点击**保存并测试**:
    *   **SMTP 主机**
    *   **SMTP 端口**
    *   **使用 SSL/TLS** -强制安全 SSL 和 TLS 协议。如果禁用，八达通仍然使用这个选项，如果你的服务器支持任一协议。
    *   **发件人地址** -你希望八达通卡发送的地址。
    *   **凭证** -电子邮件帐户的用户名和密码。

如果测试失败，请检查您的详细信息并重试。一些电子邮件服务，如 Gmail，可能需要更改设置以允许外部应用程序发送电子邮件。

## 简单的支持邮件

如果您的步骤或脚本足够简单，问题很容易解决，电子邮件提醒可能就足够了。在这种情况下，创建自动化的支持电子邮件步骤非常简单。

我们在示例实例中创建了一个简单的 runbook，它运行一个注定要失败的“Hello World！”剧本。失败时，runbook 会向支持地址发送一封电子邮件。

像这样增加一个步骤会让那些能够修复它们的人注意到过时和破损的运行手册。

下面是我们如何设置**发送电子邮件**步骤:

1.  打开您想要触发支持电子邮件的项目。
2.  点击**操作**。
3.  点击**运行手册**。
4.  点击现有的运行手册进行编辑，或使用**添加运行手册**按钮创建新的运行手册。如果创建新的运行手册，输入名称和描述，然后点击**保存**。
5.  点击**添加步骤**。
6.  搜索`email`，悬停在**上，从结果中发送电子邮件**，点击**添加**。
7.  填写以下字段并点击**保存**:
    *   **步骤名称** -给步骤起一个描述性的名称。
    *   **至**、**抄送**和**密件抄送** -输入电子邮件地址或从下拉列表中选择一个团队。关于[创建团队](https://octopus.com/docs/security/users-and-teams)的更多信息，请参见我们的文档。
    *   **主题**
    *   正文**——使用原始文本或 HTML 格式的邮件。**
    *   **运行条件** -如果警告某人部署或运行手册步骤失败，选择**失败:仅在前一步骤失败时运行**。
    *   **启动触发器** -通常最好离开**等待前一步完成，然后启动**。

当创建或编辑一本运行手册时，您必须点击**发布**以使其对其他团队和 Octopus 触发事件可用。

新步骤总是出现在 runbook 流程的底部。如果您需要对步骤重新排序:

1.  打开你的跑步手册。
2.  转到**进程**选项卡。
3.  单击任何步骤打开流程编辑器。
4.  点击**按名称过滤**搜索框旁边的菜单按钮(3 个垂直点)并选择**重新排序步骤**。
5.  将您的步骤拖至您需要的顺序，然后单击**完成**。

## 高级支持电子邮件

对于容易解决的问题，一封简单的支持邮件就可以了，但是如果您的支持团队需要更多信息呢？

有了[输出变量](https://octopus.com/docs/projects/variables/output-variables)和一点努力，**发送电子邮件**步骤也可以包括开始故障诊断的一切。

在本例中，我们的[高级示例操作手册](https://tenpillars.octopus.app/app#/Spaces-103/projects/advanced-email-example/operations/runbooks)从 Kubernetes 集群中抓取信息，以便在电子邮件中发送，包括:

*   部署信息
*   Pod 日志
*   部署目标的描述

这种类型的操作手册非常适合:

*   从您的部署目标中自动提取技术信息
*   帮助不太懂技术的员工在不知道复杂术语或命令的情况下获取信息
*   加速系统恢复

### 开始之前

该示例使用:

*   Kubernetes 作为部署目标，尽管您可以为其他目标类型定制这个想法。
*   PowerShell 脚本从 Kubernetes 抓取数据。您可能需要在 Linux 发行版上安装 PowerShell，这些脚本才能工作。参见[微软的 PowerShell 安装文档](https://docs.microsoft.com/en-us/powershell/scripting/install/installing-powershell?view=powershell-7.2)了解如何安装。

### 创建项目操作手册

1.  打开您想要触发支持电子邮件的项目。
2.  点击**操作**。
3.  点击**运行手册**。
4.  点击**添加 RUNBOOK** 。
5.  输入名称和描述，点击**保存**。

### 创建 Kubernetes 步骤

现在我们创建从 Kubernetes 集群中获取信息的步骤。这些步骤使用输出变量，因此我们可以将信息添加到电子邮件中。

使用**添加步骤**按钮创建 3 个 **Kubectl CLI 脚本**步骤，它们具有以下名称和 PowerShell 脚本。

要创建步骤:

1.  在新操作手册中，点击**流程**选项卡，然后**添加步骤**。
2.  点击 **Kubernetes** ，然后**在**安装的步骤模板**下运行一个 kubectl CLI 脚本**。
3.  填写以下字段并点击**保存**:
    *   **步骤名称** -从下面的步骤信息中获取步骤名称。
    *   **代表**——从下拉列表中选择您的目标角色。
    *   **工作池**——检查**是否在特定工作池**的工作机上运行，并从下拉菜单中选择**托管的 Ubuntu** 。(仅在使用 Octopus Workers 时需要。)
    *   **容器镜像** -检查**在容器内部运行，在一个工作器**上，然后点击**使用最新的基于 Linux 的镜像**来自动完成字段。(仅在使用 Octopus Workers 时需要。)
    *   **内联源代码** -检查 **Powershell** 并输入下面每个步骤的代码。

#### 步骤 1:获得部署

称这个步骤为`Get deployment`。我们在设置支持电子邮件步骤时会引用该名称。

在**内联源代码**字段中输入以下代码。将**的名称空间**换成你自己集群的名称空间。

```
ps
$output = (kubectl describe deployment -n) -join "`n"
Set-OctopusVariable -name "Describe[$($_.metadata.name)].Content" -value $output

exit 0 
```

#### 步骤 2:获取 pod 日志

把这个步骤叫做`Get pod logs`。我们在设置支持电子邮件步骤时会引用该名称。

在**内联源代码**字段中输入以下代码，并替换**部署名称**您自己的 Kubernete 部署名称。

```
ps
$pods = kubectl get pods -o json | ConvertFrom-Json
$items = if ($pods.items -ne $null) {$pods.items} else {@($pods)}

$items | 
    ? {$_.metadata.name -like "deployment-name*"} |
    % {
        $logs = (kubectl logs $_.metadata.name) -join "`n"
        Set-OctopusVariable -name "Logs[$($_.metadata.name)].Content" -value $logs
    }

exit 0 
```

#### 第三步:获取描述

这一步叫做`Get description`。我们在设置支持电子邮件步骤时会引用该名称。

在**内联源代码**字段中输入以下代码，并替换**部署名称**您自己的 Kubernete 部署名称。

```
ps
$pods = kubectl get pods -o json | ConvertFrom-Json
$items = if ($pods.items -ne $null) {$pods.items} else {@($pods)}

$items | 
    ? {$_.metadata.name -like "deployment-name*"} |
    % {
        $logs = (kubectl describe pod $_.metadata.name) -join "`n"
        Set-OctopusVariable -name "Describe[$($_.metadata.name)].Content" -value $logs
    }

exit 0 
```

### 创建电子邮件步骤

现在，我们添加了**发送电子邮件**步骤，并将其配置为包含收集的信息。

1.  点击**添加步骤**。
2.  搜索`email`，悬停在**上，从结果中发送电子邮件**，点击**添加**。
3.  填写以下字段并点击**保存**:
    *   **步骤名称** -给步骤一个描述性名称。
    *   **至**、**抄送**和**密件抄送** -输入电子邮件地址或从下拉列表中选择一个团队。有关[创建团队](https://octopus.com/docs/security/users-and-teams)的更多信息，请参见我们的文档。
    *   **主题**
    *   **正文** -使用以下原始文本:

```
Issue context:

The description of the deployment is included below:

#{each deployment in Octopus.Action[Get deployment].Output.Describe}
Description of #{deployment}
#{deployment.Content}

-----------------------------------------------------------

#{/each}

The pod logs are included below:

#{each pod in Octopus.Action[Get pod logs].Output.Logs}
Logs for pod #{pod}
#{pod.Content}

-----------------------------------------------------------

#{/each}

The pod descriptions are included below:

#{each pod in Octopus.Action[Get description].Output.Describe}
Description of #{pod}
#{pod.Content}

-----------------------------------------------------------

#{/each} 
```

### 查看支持电子邮件

查看由我们的示例触发的电子邮件，我们可以发现 Kubernetes 集群的几个问题:

*   端口号不是一个数字
*   该窗格具有未定义的 URL

这意味着支持团队很快就知道了部署目标的问题。

```
Issue context:

The description of the deployment is included below:

Description of
Name:                   random-quotes
Namespace:              default
CreationTimestamp:      Fri, 25 Mar 2022 02:59:41 +0000
Labels:                 Octopus.Action.Id=6f79e0b9-ec3d-4071-8d42-54dbc9d5ee1b
                        Octopus.Deployment.Id=deployments-1426
                        Octopus.Deployment.Tenant.Id=untenanted
                        Octopus.Environment.Id=environments-101
                        Octopus.Kubernetes.DeploymentName=random-quotes
                        Octopus.Kubernetes.SelectionStrategyVersion=SelectionStrategyVersion2
                        Octopus.Project.Id=projects-162
                        Octopus.RunbookRun.Id=
                        Octopus.Step.Id=b9777e71-4818-482e-8cb8-79ebf9b9960b
Annotations:            deployment.kubernetes.io/revision: 2
Selector:               Octopus.Kubernetes.DeploymentName=random-quotes
Replicas:               1 desired | 1 updated | 1 total | 1 available | 0 unavailable
StrategyType:           RollingUpdate
MinReadySeconds:        0
RollingUpdateStrategy:  25% max unavailable, 25% max surge
Pod Template:
  Labels:  Octopus.Action.Id=6f79e0b9-ec3d-4071-8d42-54dbc9d5ee1b
           Octopus.Deployment.Id=deployments-1426
           Octopus.Deployment.Tenant.Id=untenanted
           Octopus.Environment.Id=environments-101
           Octopus.Kubernetes.DeploymentName=random-quotes
           Octopus.Kubernetes.SelectionStrategyVersion=SelectionStrategyVersion2
           Octopus.Project.Id=projects-162
           Octopus.RunbookRun.Id=
           Octopus.Step.Id=b9777e71-4818-482e-8cb8-79ebf9b9960b
  Containers:
   octopussamples:
    Image:      index.docker.io/octopussamples/randomquotesnodejs:1.0.2
    Port:       8080/TCP
    Host Port:  0/TCP
    Environment:
      PORT:  notanumber
    Mounts:  <none>
  Volumes:   <none>
Conditions:
  Type           Status  Reason
  ----           ------  ------
  Available      True    MinimumReplicasAvailable
  Progressing    True    NewReplicaSetAvailable
OldReplicaSets:  <none>
NewReplicaSet:   random-quotes-5db56b95dc (1/1 replicas created)
Events:          <none>

-----------------------------------------------------------

The pod logs are included below:

Logs for pod random-quotes-5db56b95dc-xvhnn
App listening at http://undefined:undefined

-----------------------------------------------------------

The pod descriptions are included below:

Description of random-quotes-5db56b95dc-xvhnn
Name:         random-quotes-5db56b95dc-xvhnn
Namespace:    default
Priority:     0
Node:         aks-agentpool-33894862-vmss000001/10.240.0.5
Start Time:   Fri, 25 Mar 2022 03:07:56 +0000
Labels:       Octopus.Action.Id=6f79e0b9-ec3d-4071-8d42-54dbc9d5ee1b
              Octopus.Deployment.Id=deployments-1426
              Octopus.Deployment.Tenant.Id=untenanted
              Octopus.Environment.Id=environments-101
              Octopus.Kubernetes.DeploymentName=random-quotes
              Octopus.Kubernetes.SelectionStrategyVersion=SelectionStrategyVersion2
              Octopus.Project.Id=projects-162
              Octopus.RunbookRun.Id=
              Octopus.Step.Id=b9777e71-4818-482e-8cb8-79ebf9b9960b
              pod-template-hash=5db56b95dc
Annotations:  <none>
Status:       Running
IP:           10.244.2.11
IPs:
  IP:           10.244.2.11
Controlled By:  ReplicaSet/random-quotes-5db56b95dc
Containers:
  octopussamples:
    Container ID:   containerd://07a18a2468da4324be06c11eaeb7cdcc90d0cf67f701f7b140823cc8a7b3b80d
    Image:          index.docker.io/octopussamples/randomquotesnodejs:1.0.2
    Image ID:       docker.io/octopussamples/randomquotesnodejs@sha256:b2104dd603ef648f92cdd605ecf814e4cb24a2e1827f036b004985d06a380644
    Port:           8080/TCP
    Host Port:      0/TCP
    State:          Running
      Started:      Fri, 25 Mar 2022 03:07:56 +0000
    Ready:          True
    Restart Count:  0
    Environment:
      PORT:  notanumber
    Mounts:
      /var/run/secrets/kubernetes.io/serviceaccount from kube-api-access-v5rgh (ro)
Conditions:
  Type              Status
  Initialized       True
  Ready             True
  ContainersReady   True
  PodScheduled      True
Volumes:
  kube-api-access-v5rgh:
    Type:                    Projected (a volume that contains injected data from multiple sources)
    TokenExpirationSeconds:  3607
    ConfigMapName:           kube-root-ca.crt
    ConfigMapOptional:       <nil>
    DownwardAPI:             true
QoS Class:                   BestEffort
Node-Selectors:              <none>
Tolerations:                 node.kubernetes.io/not-ready:NoExecute for 300s
                             node.kubernetes.io/unreachable:NoExecute for 300s
Events:                      <none> 
```

## 结论

这些例子是对 Octopus Runbooks 如何帮助你获得关键信息的一个小小的了解。

如果你想亲自尝试章鱼手册，[注册免费试用](https://octopus.com/start)，体验乐趣。

阅读我们的 [Runbooks 系列](https://octopus.com/blog/tag/Runbooks%20Series)的其余部分。

愉快的部署！