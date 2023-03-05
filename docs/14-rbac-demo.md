# 基于角色的访问控制演示- Octopus 部署

> 原文：<https://octopus.com/blog/k8s-training/14-rbac-demo>

这篇文章是我们 Kubernetes 培训系列的第 14 篇，为 DevOps 工程师提供了关于 Docker、Kubernetes 和 Octopus 的介绍。

此视频演示了如何创建一个 Octopus 目标，该目标使用服务帐户令牌向集群进行身份验证。

如果您还没有八达通帐户，您可以开始免费试用。

您可以使用下面的链接完成该系列。

## 示例代码

### RBAC 资源公司

这是复合 YAML 文档，包含用于将服务帐户限制到单个名称空间的 RBAC 资源:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: octopub-deployer
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: octopub-deployer-role
rules:
- apiGroups: ["", "extensions", "apps", "networking.k8s.io"]
  resources: ["deployments", "replicasets", "pods", "services", "ingresses", "secrets", "configmaps"]
  verbs: ["get", "list", "watch", "create", "update", "patch", "delete"]
- apiGroups: [""]
  resources: ["namespaces"]
  verbs: ["get"]
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: octopub-deployer-rolebinding
subjects:
- kind: ServiceAccount
  name: octopub-deployer
  apiGroup: ""
roleRef:
  kind: Role
  name: octopub-deployer-role
  apiGroup: ""
---
apiVersion: v1
kind: Secret
type: kubernetes.io/service-account-token
metadata:
  name: octopub-deployer-secret
  annotations:
    kubernetes.io/service-account.name: "octopub-deployer" 
```

### 目标创建脚本

该脚本通过从当前 Kubernetes 上下文中提取密码和 Kubernetes URL 来创建新的 Octopus 令牌帐户和目标:

```
SERVER=$(kubectl config view -o json | jq -r '.clusters[0].cluster.server')
TOKEN=$(kubectl get secret octopub-deployer-secret -n octopub -o json | jq -r '.data.token' | base64 -d)

echo "##octopus[create-tokenaccount \
  name=\"$(encode_servicemessagevalue "Octopub #{Octopus.Environment.Name}")\" \
  token=\"$(encode_servicemessagevalue "${TOKEN}")\" \
  updateIfExisting=\"$(encode_servicemessagevalue 'True')\"]"

echo "##octopus[create-kubernetestarget \
  name=\"$(encode_servicemessagevalue "Octopub #{Octopus.Environment.Name}")\" \
  octopusRoles=\"$(encode_servicemessagevalue 'Octopub')\" \
  clusterUrl=\"$(encode_servicemessagevalue "${SERVER}")\" \
  octopusAccountIdOrName=\"$(encode_servicemessagevalue "Octopub #{Octopus.Environment.Name}")\" \
  namespace=\"$(encode_servicemessagevalue "octopub")\" \
  octopusDefaultWorkerPoolIdOrName=\"$(encode_servicemessagevalue "Laptop")\" \
  updateIfExisting=\"$(encode_servicemessagevalue 'True')\" \
  skipTlsVerification=\"$(encode_servicemessagevalue 'True')\"]" 
```

## 资源

## 了解更多信息

如果您想在 AWS 平台(如 EKS 和 ECS)上构建和部署容器化的应用程序，请尝试使用 [Octopus Workflow Builder](https://octopusworkflowbuilder.octopus.com/#/) 。构建器使用 GitHub Actions 工作流构建的示例应用程序填充 GitHub 存储库，并使用示例部署项目配置托管的 Octopus 实例，这些项目展示了最佳实践，如漏洞扫描和基础架构代码(IaC)。

愉快的部署！