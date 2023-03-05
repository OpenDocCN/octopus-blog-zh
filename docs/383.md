# 创建 Kubernetes pods、复制集和部署- Octopus 部署

> 原文：<https://octopus.com/blog/k8s-training/4-creating-kubernetes-resources>

这篇文章是我们 Kubernetes 培训系列的第四篇，为 DevOps 工程师提供了关于 Docker、Kubernetes 和 Octopus 的介绍。

此视频演示了 Kubernetes pods、副本集和部署，并展示了每种部署的示例。

如果您还没有八达通帐户，您可以开始免费试用。

您可以使用下面的链接完成该系列。

## 资源

### 样品舱 YAML

```
apiVersion: v1
kind: Pod
metadata:
  name: underwater
spec:
  containers:
  - name: webapp
    image: octopussamples/underwater-app
    ports:
    - containerPort: 80 
```

### 样本复制集 YAML

```
apiVersion: apps/v1
kind: ReplicaSet
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: webapp
  template:
    metadata:
      labels:
        tier: webapp
    spec:
      containers:
      - name: webapp
        image: octopussamples/underwater-app
        ports:
        - containerPort: 80 
```

### 示例部署 YAML

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: webapp
spec:
  replicas: 3
  selector:
    matchLabels:
      tier: webapp
  template:
    metadata:
      labels:
        tier: webapp
    spec:
      containers:
      - name: webapp
        image: octopussamples/underwater-app
        ports:
        - containerPort: 80 
```

## 了解更多信息

如果您希望在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，那么 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 将用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础设施代码(IaC)。

愉快的部署！