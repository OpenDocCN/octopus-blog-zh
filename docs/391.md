# 集成 OAuth 和 Kubernetes - Octopus 部署

> 原文：<https://octopus.com/blog/kubernetes-oauth>

[![Kubernetes OAuth illustration showing a locked treasure chest representing Kubernetes](img/8f6ddec414242ddbd59e46f72366beda.png)](#)

对于组织中的任何基础设施来说，管理断开连接的用户数据库都是一个主要的难点，更不用说安全漏洞了。Kubernetes 也不例外，因为默认情况下，系统的用户是特定于 Kubernetes 本身的。

这个问题的一个常见解决方案是允许用户通过 OAuth 向 Kubernetes 进行身份验证，这意味着可以使用现有的登录提供商(如 Google 或 Microsoft)来验证用户凭证。

在这篇博文中，我们将看看如何将 Minikube 与 Google 集成，以在 Kubernetes 中提供基于浏览器的登录。

## 创建 OAuth 客户端

第一步是在 Google 中创建一个 OAuth 客户端。打开[https://console.cloud.google.com/apis/credentials](https://console.cloud.google.com/apis/credentials)，从下拉列表中选择一个项目，或者创建一个新项目。接下来，从**创建凭证**下拉列表中选择 **OAuth 客户端 ID** 选项:

[![](img/6535c00ab852e9cb9d3c8d69eb0ec48a.png)](#)

选择**其他**选项，将客户端名称设置为 **minikube** :

[![](img/9b26daa709daeb1f66db579a9fe2256a.png)](#)

你现在会看到两个代码:客户端 ID 和客户端密码**。记下这两个代码，因为我们稍后会用到它们:**

[![](img/86967b30091f0c1ec992550a2c7f605b.png)](#)

生成这些代码后，我们就可以启动 Minikube 了。

## 使用 OAuth 详细信息配置 Minikube

要允许 Minikube 接受 Google 登录，我们需要传递以下参数:

*   `--extra-config=apiserver.authorization-mode=RBAC`
*   `--extra-config=apiserver.oidc-issuer-url="https://accounts.google.com"`
*   `--extra-config=apiserver.oidc-client-id=<Client ID>`
*   `--extra-config=apiserver.oidc-username-claim=email`

这些值启用 RBAC 安全性，配置我们刚刚创建的 OAuth 客户端 ID，并指定 Google 用户的电子邮件地址成为他们的 Kubernetes 用户名。

因为我运行的是 Windows 和 PowerShell，所以我的 Minikube start 命令如下所示:

```
minikube start `
--extra-config=apiserver.authorization-mode=RBAC `
--extra-config=apiserver.oidc-issuer-url="https://accounts.google.com" `
--extra-config=apiserver.oidc-client-id=471129667683-049fg2q12m993hk8c5hq1jhf0ji1ske5.apps.googleusercontent.com `
--extra-config=apiserver.oidc-username-claim=email `
--vm-driver hyperv `
--hyperv-virtual-switch "External Switch" 
```

您应该会看到如下所示的输出:

```
PS C:\Users\Matthew> minikube start --extra-config=apiserver.authorization-mode=RBAC `
>> --extra-config=apiserver.oidc-issuer-url="https://accounts.google.com" `
>> --extra-config=apiserver.oidc-client-id=471129667683-049fg2q12m993hk8c5hq1jhf0ji1ske5.apps.googleusercontent.com `
>> --extra-config=apiserver.oidc-username-claim=email `
>> --vm-driver hyperv `
>> --hyperv-virtual-switch "External Switch"
* minikube v1.3.1 on Microsoft Windows 10 Pro 10.0.18362 Build 18362
* Creating hyperv VM (CPUs=2, Memory=2000MB, Disk=20000MB) ...
* Preparing Kubernetes v1.15.2 on Docker 18.09.8 ...
  - apiserver.authorization-mode=RBAC
  - apiserver.oidc-issuer-url=https://accounts.google.com
  - apiserver.oidc-client-id=471129667683-8bsdnomeq3autchmmvut6949hrh74mpg.apps.googleusercontent.com
  - apiserver.oidc-username-claim=email
* Downloading kubeadm v1.15.2
* Downloading kubelet v1.15.2
* Pulling images ...
* Launching Kubernetes ...
* Waiting for: apiserver proxy etcd scheduler controller dns
* Done! kubectl is now configured to use "minikube" 
```

此时，Minikube 已经用管理员凭证配置了一个本地`~/.kube/config`文件，所以我们可以直接使用`kubectl`。但是，任何操作都将以本地 Kubernetes 管理员的身份执行，而不是以 Google 用户的身份。不过，这没问题，因为我们还有一些额外的配置要做:

```
PS C:\Users\Matthew> kubectl get nodes
NAME       STATUS   ROLES    AGE     VERSION
minikube   Ready    master   4m49s   v1.15.2 
```

## 添加集群角色

在为我们的新 Google 用户做准备时，我们需要定义一个`ClusterRole`来授予他们管理员访问权限，并定义一个`ClusterRoleBinding`来将用户绑定到角色。

这里是`ClusterRole` YAML，它让用户管理员可以访问 Minikube 集群:

```
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-role
rules:
- apiGroups: ["*"]
  resources: ["*"]
  verbs: ["*"] 
```

下面是`ClusterRoleBinding` YAML，它将新用户(在本例中我是`matthew.casperson@octopus.com`)映射到我们上面定义的角色:

```
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: admin-binding
subjects:
- kind: User
  name: matthew.casperson@octopus.com
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: admin-role 
```

将这些文件的内容保存到`clusterrole.yaml`和`clusterrolebinding.yaml`，然后用命令应用它们:

```
kubectl apply -f clusterrole.yaml
kubectl apply -f clusterrolebinding.yaml 
```

## 生成 OAuth 令牌

您可能已经熟悉 OAuth 登录，因为它们通常用于在 web 应用程序中对用户进行身份验证。

然而，从控制台应用程序对用户进行身份验证略有不同。我们仍然需要用户打开一个网页并用 Google 验证自己，但是`kubectl`本身不会与 web 浏览器交互，所以我们需要一些其他的方法来生成这些代码。

这就是 [k8s-oidc-helper](https://github.com/micahhausler/k8s-oidc-helper) 的用武之地。这个工具生成一个我们可以在浏览器中打开的 URL。URL 显示所需的 Google 令牌，然后我们可以将其粘贴回控制台，并且`k8s-oidc-helper`生成`kubectl`验证用户所需的代码。让我们看看这个过程是如何进行的。

要安装`k8s-oidc-user`，确保安装了 [Go 工具](https://golang.org/doc/install)，然后运行:

```
go get github.com/micahhausler/k8s-oidc-helper 
```

一旦安装完毕，我们就可以使用前一步生成的**客户端 ID** 和**客户端秘密**运行`k8s-oidc-helper`:

```
k8s-oidc-helper --client-id 471129667683-049fg2q12m993hk8c5hq1jhf0ji1ske5.apps.googleusercontent.com --client-secret Cz2FbfSsue2RI_KKd2EawEjG 
```

系统将提示您打开一个 URL:

```
PS C:\Users\Matthew\Desktop> k8s-oidc-helper --client-id 471129667683-049fg2q12m993hk8c5hq1jhf0ji1ske5.apps.googleusercontent.com --client-secret Cz2FbfSsue2RI_KKd2EawEjG                                                         
Open this url in your browser: https://accounts.google.com/o/oauth2/auth?redirect_uri=urn:ietf:wg:oauth:2.0:oob&response_type=code&client_id=471129667683-049fg2q12m993hk8c5hq1jhf0ji1ske5.apps.googleusercontent.com&scope=openid+email+profile&approval_prompt=force&access_type=offline          

Enter the code Google gave you: 
```

打开网址，出现熟悉的谷歌登录页面。一旦您确认了您的帐户详细信息，Google 将提供一个代码粘贴回控制台:

[![](img/bfbda83ae597cc96f26b2f4eb2a74305.png)](#)

然后，`k8s-oidc-helper`会给你一份部分完整的`~/.kube/config`文件副本，供你保存在本地。

在`users`部分下的代码就是我们所需要的，所以替换`~/.kube/config`文件中现有的`users`部分，并将`contexts.user`参数更改为新的用户名。您将得到一个类似这样的`~/.kube/config`文件:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: C:\Users\Matthew\.minikube\ca.crt
    server: https://10.1.1.94:8443
  name: minikube
contexts:
- context:
    cluster: minikube
    # Make sure this is the same as the user below
    user: matthew.casperson@octopus.com
  name: minikube
current-context: minikube
kind: Config
preferences: {}
users:
# This section came from k8s-oidc-helper
- name: matthew.casperson@octopus.com
  user:
    auth-provider:
      config:
        client-id: 471129667683-049fg2q12m993hk8c5hq1jhf0ji1ske5.apps.googleusercontent.com
        client-secret: Cz2FbfSsue2RI_KKd2EawEjG
        id-token: eyJhbGciOiJSUzI1NiIsImtpZCI6ImY2ZjgwZjM3ZjIxYzIzZTYxZjJiZTQyMzFlMjdkMjY5ZDY2OTUzMjkiLCJ0eXAiOiJKV1QifQ.eyJpc3MiOiJhY2NvdW50cy5nb29nbGUuY29tIiwiYXpwIjoiNDcxMTI5NjY3NjgzLTA0OWZnMnExMm05OTNoazhjNWhxMWpoZjBqaTFza2U1LmFwcHMuZ29vZ2xldXNlcmNvbnRlbnQuY29tIiwiYXVkIjoiNDcxMTI5NjY3NjgzLTA0OWZnMnExMm05OTNoazhjNWhxMWpoZjBqaTFza2U1LmFwcHMuZ29vZ2xldXNlcmNvbnRlbnQuY29tIiwic3ViIjoiMTAyNjAzOTY1MzQzNTAzMjY0MDc5IiwiaGQiOiJvY3RvcHVzLmNvbSIsImVtYWlsIjoibUVFYufyutfBV5jYXNwZXJzb25Ab2N0b3B1cy5jb20iLCJlbWFpbF92ZXJpZmllZCI6dHJ1ZSwiYXRfaGFzaCI6InlMSjhGRHZVdjhQVUktVDduN0hRSEEiLCJpYXQiOjE1NjcwNDk1MTIsImV4wegvfiye2NzA1MzExMn0.Rhz_aiPvUVt3kotiv6nPeqK0JYJHjaOaaCPhMpEuEWUcNNaQjmFhIrMvbRYggtvSSnD7NYlrY02fTl9XfeBCssqhMcNpYpzehO08844w7_mjPZOLPRygZrVnWWTHvMdIHk4_oolDBYA1w6_whJ7T2ZTlRViEJJMEwkTaRCaG8BXdNg0CgvzhCpNB7BC-Dv9Xc2hvRmkzwZBJyDbtnkFfeDX9TDrQtdwbKtrhjWlTFdtlq8o0lJQKSfXRrL6fF36kQvWwSTHbB8GSfAZESsT6yaBfhKHsoByzahPw5uYjt3n5dtOoUt8kVgJfSTBzNedpsiVwpNM1YLzaBJQUGYNypQ
        idp-issuer-url: https://accounts.google.com
        refresh-token: 1/_lGBydf3ymsn4ybwrLZWqPVNtggmDFgrYIqDfKrpSzVK0ZjkigfuNX2ze9cyNc3-b
      name: oidc 
```

此时，当您使用`kubectl`时，您将被认证为 Google 用户。

请注意，任何潜在的 Google 用户都可以生成使用`kubectl`的代码，但是如果没有相应的`ClusterRoleBinding`，这些用户将实际上无权做任何事情。

## 结论

将 Kubernetes 与 OAuth 提供者集成在一起，就不需要再维护另一个由 Kubernetes 自己管理的断开连接的用户数据库，而且它允许您利用已经配置好的任何现有帐户提供者。`k8s-oidc-user`提供了一种生成所需代码的方法，之后`kubectl`就像对任何其他 Kubernetes 帐户一样工作。