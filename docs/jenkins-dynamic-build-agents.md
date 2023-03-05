# 在 Jenkins - Octopus 部署中使用动态构建代理来自动扩展

> 原文：<https://octopus.com/blog/jenkins-dynamic-build-agents>

如果你在运行一个只有几个开发人员的小项目，那么 Jenkins 的一个实例是不错的。但是随着您的团队和产品的成长，您会发现单个实例可能无法保持稳定。当提交数量增加时，Jenkins 需要运行的流程也会增加，单个实例的性能很快就会下降，并使您的团队慢下来。

谢天谢地，Jenkins 是一个可扩展的平台。可扩展性意味着，随着您处理需求的增长，Jenkins 也可以随之增长。

Jenkins 的可伸缩性将一个实例视为一个控制器，将作业定向到称为代理的其他实例。控制器知道每个代理的容量，并会将您的构建和测试发送到当时最合适的代理。通过使用动态构建代理，这个过程可以自动发生，允许 Jenkins 对您的需求做出反应。

感谢像 Kubernetes 和 Amazon Web Services (AWS)这样的虚拟环境，您不需要数量有限的代理或物理硬件。动态设置中的 Jenkins 足够聪明:

*   如果没有合适的代理，则增加新的代理
*   清理未使用的代理
*   替换损坏的安装

而且完全不需要人工干预。

在这篇文章中，我们从头到尾看了两种设置动态伸缩的流行方法，分别是 [Kubernetes](#method1) 和 [Amazon Web Services (AWS)](#method2) 。

## 方法 1:使用 Kubernetes 进行扩展

Kubernetes 是一个工具，它可以自动调整保持应用程序平稳运行所需的容器数量。这使得它成为帮助扩展 Jenkins 实例的好选择。

通过将您的 Jenkins 控制器部署到 Kubernetes，您的持续集成(CI)设置变得更易于管理、复制或在出现问题时重新创建。

容器是不太复杂的虚拟机，很容易部署到大多数操作系统或云服务。

### 开始之前

本指南只是一个示例，在更改现有的 Jenkins 设置之前，您应该尝试缩放。

在本例中，您在本地 minikube 集群上设置了可伸缩性，并使用下面的工具进行配置。如果您正在跟进，请按照列出的顺序安装工具:

1.  Docker 桌面–只有在 Windows 上才需要。确保 Docker Desktop 被设置为管理 Linux 容器而不是 Windows 容器。
2.  [minikube](https://minikube.sigs.k8s.io/docs/start/)–允许您在计算机上安装 Kubernetes 集群。
3.  [chocolate y](https://chocolatey.org/)–只有在 Windows 上才需要。它是一个命令行软件管理包，用于安装 Kubectl。
4.  [ku bectl](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/#install-on-windows-using-chocolatey-or-scoop)–控制 Kubernetes 集群的命令行工具。使用这个 Chocolatey 命令行来安装 Kubectl: `choco install kubernetes-cli`

您可以使用您习惯的任何工具来设置可伸缩性，但是您可能需要稍微调整我们的说明。

### 步骤 1:创建一个 Jenkins 控制器映像

首先，您必须创建一个 dockerfile 文件并使用它来构建一个 Jenkins 控制器映像。

dockerfile 是一个文本文件，用于在 Docker 中创建图像，然后您可以将它推送到 Docker Hub。

我们的示例 dockerfile 将创建一个包含 Jenkins 以及 Blue Ocean 和 Kubernetes 插件的图像。

要创建 dockerfile 文件并构建映像:

1.  创建一个名为`Dockerfile`的文本文件，并添加以下 Jenkins 建议的脚本。如果你需要的话，你可以在列表中添加更多的插件——只需添加它们的名字，用空格分开:

    ```
    FROM jenkins/jenkins:lts-slim  # Pipelines with Blue Ocean UI and Kubernetes RUN jenkins-plugin-cli --plugins blueocean kubernetes 
    ```

2.  保存并关闭文件。如果在 Windows 上使用记事本，您必须删除。' txt '文件扩展名。
3.  打开终端，使用 CD 命令移动到包含该文件的文件夹。使用以下命令创建镜像:

    ```
    docker build . -t [username]/jenkinsdockerfile 
    ```

4.  构建需要一点时间来处理，但是完成后你会在 Docker 桌面上或者用`docker images`命令看到图像。

当您在下一步中创建 minikube 群集时，它不会看到本地存储在您的计算机上的映像，因为群集运行在虚拟环境中。要解决这个问题，您可以将图像推送到 Docker Hub。

如果在 Windows 上，您可以在 Docker 桌面中完成此操作:

1.  点击 Docker 桌面左侧的**图片**。
2.  将光标悬停在您的图像上，单击菜单图标(3 个垂直点)并选择**按键控制**。

或者，您可以在终端中使用以下命令行:

```
Docker push -t [username]/[image name] 
```

### 步骤 2:创建一个 Kubernetes 集群

打开一个终端窗口，使用以下命令创建一个 Kubernetes 集群:

```
minikube start 
```

设置群集可能需要一段时间。完成后，给它一个新的名称空间，使集群更容易使用和引用。在终端中使用以下命令创建名称空间“jenkins”:

```
kubectl create namespace jenkins 
```

### 步骤 3:使用 YAML 部署文件安装 Jenkins

将下面的代码复制到一个文本文件中，保存为`jenkins.yaml`。确保将**图像**行更改为指向您的图像，例如【Docker 用户名】/【图像名称】。

```
# Service account docs:
# https://support.cloudbees.com/hc/en-us/articles/360038636511-Kubernetes-Plugin-Authenticate-with-a-ServiceAccount-to-a-remote-cluster
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: jenkins
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: jenkins
rules:
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/exec"]
  verbs: ["create","delete","get","list","patch","update","watch"]
- apiGroups: [""]
  resources: ["pods/log"]
  verbs: ["get","list","watch"]
- apiGroups: [""]
  resources: ["events"]
  verbs: ["watch"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get"]

---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: jenkins
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: jenkins
subjects:
- kind: ServiceAccount
  name: jenkins
---
apiVersion: apps/v1 
kind: Deployment 
metadata: 
  name: jenkins 
spec: 
  replicas: 1 
  selector: 
    matchLabels: 
      app: jenkins 
  template:
    metadata:
      labels:
        app: jenkins
    spec:
      containers:
      - name: jenkins 
        image: [docker username]/jenkins 
        ports:
        - containerPort: 8080
        - containerPort: 50000
        volumeMounts: 
        - name: jenkins-home
          mountPath: /var/jenkins_home 
      volumes:
      - name: jenkins-home 
        emptyDir: { } 
      serviceAccountName: jenkins
---
apiVersion: v1 
kind: Service 
metadata: 
  name: jenkins 
spec: 
  type: NodePort 
  ports: 
  - port: 8080 
    targetPort: 8080 
  selector: 
    app: jenkins
---
apiVersion: v1
kind: Service
metadata:
  name: jenkins-agent
spec:
  selector: 
    app: jenkins
  type: NodePort  
  ports:
    - port: 50000
      targetPort: 50000 
```

要将映像部署到命名空间，请从文件的目录中运行以下命令:

```
kubectl apply -f jenkins.yaml -n jenkins jenkins.yaml 
```

YAML 脚本现在将在 Kubernetes pod 中创建一个 Jenkins 实例。这个过程可能需要一段时间，但是您可以使用下面的命令来检查进度。当**状态**栏显示为**运行**时，表示准备就绪；

```
kubectl get pods -n jenkins 
```

### 步骤 4:找到您的 Jenkins 实例 URL 并连接到它

运行以下命令找出实例的 IP 地址，并记下它:

```
minikube ip 
```

现在我们需要发现端口。运行以下命令:

```
kubectl get services -n jenkins 
```

**端口**列将以 8080:54321 格式显示实例的端口。记下冒号后的 5 个数字。

现在，您可以将 minikube IP 和端口结合起来组成 URL。例如，如果 IP 是 123.123.123.123，端口号是 54321，那么您的 Jenkins URL 将是 http://123.123.123.123:54321。

如果您的 URL 不起作用，可能是您的防火墙阻止了该实例。请向网络管理员寻求帮助。如果您在自己的计算机上遇到这个问题，您可以使用`kubectl port-forward svc/jenkins -n jenkins 8080:8080`暂时转发您的端口。这将使您的 URL http://localhost:8080。

转到您的网络浏览器中的 URL，您应该会看到**入门**屏幕。这将要求一次性管理员密码。对于 Kubernetes 集群，您需要通过命令行找到它。

首先，您需要 Kubernetes pod 的名称。使用以下命令:

```
kubectl get pods -n Jenkins 
```

你的 pod 名字看起来会像这样:**Jenkins-27 BC 5 DCD 98-xk9mp**。

现在运行下面的命令，用您刚刚记下的名字替换`[podname]`。

```
kubectl logs [podname] -n jenkins 
```

滚动结果，找到由星号分隔的管理员密码。将它粘贴到您的 Jenkins 实例中，您就可以完成安装了。参见 [Jenkins 文档](https://www.jenkins.io/doc/)了解如何首次设置 Jenkins。

### 步骤 5:获取最终信息并安装 Jenkins 插件

现在你可以在 Jenkins 中设置插件了。在 web 浏览器中返回 Jenkins:

1.  从菜单中点击**管理詹金斯**。
2.  点击**管理节点和云**。
3.  点击**配置云**。
4.  从下拉列表中选择 **Kubernetes** ，然后点击 **Kubernetes 云详情**。
5.  填写以下字段并点击**保存**:
    *   **Kubernetes 网址**–输入`https://kubernetes.default`
    *   **Kubernetes 名称空间**–输入`jenkins`
    *   **詹金斯网址**–输入`http://jenkins.jenkins.svc.cluster.local:8080`
    *   **詹金斯隧道**–回车`jenkins-agent.jenkins.svc.cluster.local:50000`
6.  滚动到底部，点击 **Pod 模板**，然后**添加 Pod 模板**，以及 **Pod 模板详情**。
7.  填写以下字段并点击**保存**:
    *   **名称**–输入`jenkins-agent`
    *   **名称空间**–输入`jenkins`
    *   **标签**–输入`jenkins-agent`
    *   **用途** -从下拉列表中选择**尽可能使用该节点**

### 第六步:测试一切正常

为了测试 Jenkins 是否能够适当地伸缩，您可以创建一些简单的构建作业来检查它们是如何分布的。

首先，设置 Jenkins，使其不会在控制器上运行作业(除非您另外告诉它):

1.  从菜单中点击**管理詹金斯**。
2.  点击**配置系统**。
3.  从**用法**下拉菜单中选择**仅构建标签表达式与该节点**匹配的作业，然后点击**保存**。

然后创建 2 个“Hello World”构建作业:

1.  在 Jenkins 仪表盘上点击**新项目**。
2.  输入合适的名称，如`Testing 1`，选择**自由式项目**，点击**确定**。
3.  在**构建**标题下，从下拉框中选择**执行 shell** 。
4.  在**命令框**中输入`echo "Hello World"`，点击**保存**:
5.  重复这些步骤，但是把你的第二份工作叫做`Testing 2`。

同时运行两个生成作业。如果工作正常，它们会出现在 Jenkins 左侧的**构建队列**中。在构建作业期间，它们出现在标题为**构建执行器状态**的标题下，带有我们之前设置的“jenkins-agent”前缀。您还可以检查构建历史，仔细检查作业运行的确切位置以及它是否成功。

## 方法 2:使用 Amazon Web Services (AWS)和 EC2 Fleet 插件进行扩展

管理 Jenkins 可伸缩性的另一种方法是使用 EC2(亚马逊弹性计算云)容器和 [EC2 Fleet 插件](https://plugins.jenkins.io/ec2-fleet/)。

此选项适合以下团队:

*   已经可以访问 AWS
*   想要轻松地向现有的 Jenkins 实例添加可伸缩性
*   更喜欢使用 UI 而不是命令行来执行安装

尽管 AWS 是有成本的，但你的财务限额是在账户级别设定的。

### 配置 AWS

如果您没有帐户，请注册 AWS 如果您已经有访问权限，请登录您的帐户。

#### 步骤 1:创建策略

首先创建一个策略，允许访问使用 AWS EC2 Spot Fleet 和自动缩放组:

1.  点击顶部的**服务**菜单，选择**安全、身份、&合规**，然后选择 **IAM** 。
2.  在**访问管理**标题下的左侧菜单中点击**策略**。
3.  从右侧点击**创建策略**。
4.  使用可视化编辑器创建一个新策略，以同时使用 EC2 专色车队和自动缩放组。或者，使用 JSON 编辑器并粘贴插件设置指南中的[代码。点击**下一步:标签**。](https://plugins.jenkins.io/ec2-fleet/#plugin-content-3-configure-user-permissions)
5.  如果需要，添加标签，然后点击**下一步:查看**。
6.  为策略命名并进行描述，然后单击**创建策略**。

#### 步骤 2:创建具有编程访问权限的 IAM 用户

现在，您创建一个具有编程访问权限的用户，并为其分配新策略:

1.  点击顶部的**服务**菜单，选择**安全、身份、&合规**，然后选择 **IAM** 。
2.  在**访问管理**标题下的左侧菜单中点击**用户**。
3.  点击右侧的**添加用户**。
4.  给你的账户取一个合适的名字，选择**访问键-编程访问**，点击**下一步:权限**。
5.  点击**直接附加现有策略**按钮。搜索您在上一步中创建的策略，勾选左侧的复选框，然后单击 **Next: Tags** 。
6.  如果需要，添加标签，然后点击**下一步:查看**。
7.  查看您的新用户并点击**创建用户**。

#### 步骤 3:设置凭证以将 AWS 连接到 Jenkins

接下来，创建将 AWS 连接到 Jenkins 所需的两种身份验证方法。

首先，为新创建的 IAM 用户设置一个访问键 ID。这将允许 Jenkins 查看与您的 AWS 设置相关的信息。要设置访问密钥 ID:

1.  点击顶部的**服务**菜单，选择**安全、身份、&合规**，然后选择 **IAM** 。
2.  在**访问管理**标题下的左侧菜单中点击**用户**。
3.  搜索并单击您在步骤 2 中创建的 IAM 用户。
4.  转到**安全凭证**选项卡并点击**创建访问密钥**。
5.  记下访问密钥 ID 和秘密访问密钥(点击**显示**查看)并将其保存在安全的地方，如密码管理器。在这个阶段，您只能看到秘密访问密钥，如果丢失了，您需要创建一个新的密钥。

现在，您创建一个密钥对。这允许 Jenkins 连接到 AWS 在伸缩时将创建的实例。要设置密钥对:

1.  点击顶部的**服务**菜单，选择**计算**，点击 **EC2** 。
2.  点击**网络&安全**标题下左侧菜单中的**密钥对**。
3.  点击**创建密钥对**。
4.  完成以下选项并点击**创建密钥对**:
    *   **名称**–给密钥对起一个描述性的名称。
    *   **密钥对类型** -保留为 **RSA** 。
    *   **私钥文件格式**–选择**。pem** 。
    *   **标签**–如果需要，添加标签。
5.  私钥将自动下载到文本文件中。保管好这个文件，你以后会需要的。

#### 步骤 4:创建 AWS EC2 Spot 车队或自动伸缩组

您可以创建 EC2 Spot 车队或自动缩放组。在本例中，我们创建了一个 EC2 Spot 车队，但是如果您想了解更多关于[创建自动缩放组](https://docs.aws.amazon.com/autoscaling/ec2/userguide/GettingStartedTutorial.html)的信息，请查看 AWS 文档。

在开始之前，你也应该清楚地知道你想要使用什么亚马逊机器映像(AMI)。AMI 是一个预先构建的映像，包括操作系统和软件，您的 EC2 机群将从这些操作系统和软件创建额外的机器。

您的 AMI 映像应该包含 Java 11，因为没有它 Jenkins 将无法扩展。我们使用 AWS Marketplace 的[open JDK 11 Java 11 Ubuntu 18.04 AMI，然而，如果你需要一些特定的东西，你可以构建自己的 AMI。关于 AMIs](https://aws.amazon.com/marketplace/server/configuration?productId=dd67a7a9-d67f-4c91-a5ee-7e32da4da5c8&ref_=psb_cfg_continue) 的更多信息，请参见 [AWS 文档。](https://docs.aws.amazon.com/marketplace/latest/userguide/ami-products.html)

要创建 EC2 Spot 车队:

1.  点击顶部的**服务**菜单，选择**计算**，点击 **EC2** 。
2.  在**实例**标题下的左侧菜单中点击**现场请求**。
3.  点击**请求 Spot 实例**。
4.  至少设置以下选项，其他选项可以根据需要进行更改:
    *   **AMI**–记得选择包含 Java 11 的 AMI。
    *   **密钥对名称**–选择我们在步骤 3 中创建的密钥对名称。
    *   **保持目标产能**–勾选此框。
5.  滚动到底部，完成后点击**发射**。

AWS 需要一些时间来创建你的舰队。然后就可以配置 Jenkins 了。

### 配置 Jenkins

#### 步骤 1:在 Jenkins 中安装 EC2 Fleet 插件

要安装 EC2 Fleet 插件:

1.  从菜单中点击**管理詹金斯**。
2.  点击**管理插件**。
3.  点击**可用的**选项卡，开始在**过滤字段**中输入`EC2 Fleet`。插件应该出现在预测的搜索结果中。
4.  勾选插件左侧的复选框，然后点击**安装而不重启**。

Jenkins 将安装插件和所有依赖项，包括其他插件、扩展和 Amazon 软件开发工具包(SDK)。

如果你还没有，你也应该安装[凭证绑定插件](https://plugins.jenkins.io/credentials-binding/)。

#### 步骤 2:向 Jenkins 添加 AWS IAM 帐户和密钥对

要添加您的 AWS IAM 帐户:

1.  从菜单中点击**管理詹金斯**。
2.  向下滚动到**安全**标题并点击**管理凭证**。
3.  在 Jenkins 标题下的**商店范围内点击 **Jenkins** 。**
4.  点击**系统**标题下的**全局凭证(无限制)**。
5.  如果没有凭证，可以点击**添加一些凭证怎么样？**链接，否则从左边点击**添加凭证**。
6.  填写以下字段并点击**确定**:
    *   **种类**–从下拉菜单中选择 **AWS 凭证**。
    *   **ID**–给凭证起一个名字，用于在 Jenkins 中识别凭证。
    *   **描述**–输入有意义的描述。
    *   **访问密钥 ID**–输入您在 AWS 中创建 IAM 帐户时的访问密钥 ID。
    *   **秘密访问密钥**–输入您在 AWS 中创建 IAM 帐户时的访问字符串。

要添加您的密钥对:

1.  从菜单中点击**管理詹金斯**。
2.  向下滚动到**安全标题**并点击**管理凭证**。
3.  点击 Jenkins 范围内的商店标题下的 **Jenkins。**
4.  点击**系统**标题下的**全局凭证(无限制)**。
5.  点击左边**添加凭证**。
6.  填写以下字段并点击**确定**:
    *   **种类**–从下拉列表中选择 **SSH 用户名和私钥**。
    *   **ID**–给凭证一个名称，用于在 Jenkins 中识别凭证。
    *   **描述**–输入有意义的描述。
    *   **用户名**–该用户名用于您选择的 AMI。
    *   **私钥**–勾选**直接输入**单选按钮，点击**添加**按钮，粘贴您之前下载的密钥对文件的内容。

#### 步骤 3:将 Jenkins 连接到您的 AWS EC2 Spot 车队

现在您将 Jenkins 连接到 AWS EC2 Spot 舰队:

1.  从菜单中点击**管理詹金斯**。
2.  点击**管理节点和云**。
3.  点击**配置云**。
4.  从**添加新云**下拉列表中选择 **Amazon EC2 Fleet** 。填写以下字段并点击**保存**:
    *   **名称**–输入在 Jenkins 中使用的描述性名称。
    *   **AWS 凭证**–从 **AWS 凭证**下拉列表中选择您的访问密钥 ID。
    *   **区域**–从下拉列表中选择您的 EC2 舰队的区域。
    *   **EC2 车队**–任何连接到您的访问密钥 ID 的 EC2 Spot 车队都会出现在这里。如果凭证没有显示，您可能需要点击**测试连接**。
    *   **启动器**–通过 SSH 选择**启动代理，从两个新的下拉框中选择您在步骤 2 中添加的密钥对凭证和**非验证验证策略**。**

#### 第四步:测试一切正常

几分钟后，转到您的詹金斯仪表板。在左侧的**构建执行器状态**下，您应该看到您的控制器实例和您的新 EC2 车队。如果 EC2 机群显示任何错误，请单击其名称查看日志以确定问题。

为了测试 Jenkins 现在可以适当地伸缩，您可以创建简单的构建作业来检查它们是如何分布的。

首先，您要设置 Jenkins，这样它就不会在控制器上运行作业(除非您另外告诉它):

1.  从菜单中点击**管理詹金斯**。
2.  点击**配置系统**。
3.  从**用法**下拉菜单中选择**仅构建标签表达式与此节点**匹配的作业，然后点击**保存**。

现在创建一个“Hello World”构建作业:

1.  在 Jenkins 仪表盘上点击**新项目**。
2.  输入合适的名称，如`Testing`，选择**自由式项目**，点击**确定**。
3.  在**构建**标题下，从下拉框中选择**执行 shell** 。
4.  在**命令**框中输入`echo "Hello World"`，点击**保存**:

运行构建作业，如果工作正常，您会看到它出现在您的舰队下，在左侧的构建执行者状态中。一旦完成，您还可以检查您的构建历史，以检查作业运行的确切位置。

### 在 AWS 中更改缩放选项

在它设置和运行之后，您可以更改您希望 Jenkins 在 AWS 中扩展的方式。

1.  登录 AWS。
2.  点击顶部的**服务**菜单，选择**计算**，点击 **EC2** 。
3.  在**实例**标题下的左侧菜单中点击**现场请求**。
4.  点击您的 EC2 Spot 车队的**请求 ID** 。
5.  滚动到页面底部查看缩放选项。点击**自动缩放**标题下的**配置**按钮。
6.  根据需要更改设置并保存。

## 下一步是什么？

有关缩放詹金斯的更多信息，请通读他们的官方缩放文档。

你也可以阅读我们关于配置 Jenkins 的其他帖子:

[试试我们免费的 Jenkins 管道生成器工具](https://oc.to/JenkinsPipelineGenerator)用 Groovy 语法创建一个管道文件。这是您启动管道项目所需的一切。

## 观看我们的詹金斯管道网络研讨会

[https://www.youtube.com/embed/D_7AHTML_xw](https://www.youtube.com/embed/D_7AHTML_xw)

VIDEO

我们定期举办网络研讨会。请参见[网络研讨会第](https://octopus.com/events)页，了解有关即将举办的活动和实时流录制的详细信息。

阅读我们的[持续集成系列](https://octopus.com/blog/tag/CI%20Series)的其余部分。

愉快的部署！