# SSH 到 Kubernetes 集群——Octopus 部署

> 原文：<https://octopus.com/blog/ssh-into-kubernetes-cluster>

跳跃盒或堡垒主机是一种常见的网络策略，用于向公共互联网暴露单个安全入口点，以访问私有网络。这种单点入口让安全团队能够密切监视和控制对专用网络的网络访问。通常，bastion 主机公开了一个众所周知的远程访问服务，如 RDP 或 SSH，团队可以假设这些服务已经过广泛的审查，是值得信任的。

在本文中，我将解释如何在 Kubernetes 集群中托管 OpenSSH 服务器来执行管理任务。

## 部署 SSH 服务器

SSH 服务器长期以来一直被用来提供对 Linux 服务器的远程访问，并且将 SSH 服务器作为 Kubernetes pod 来托管相对容易。

下面显示的 YAML 文件创建了一个服务帐户，该帐户具有一个角色和角色绑定，授予对当前名称空间中公共资源的访问权限。然后，它部署一个`linuxserver/openssh-server`映像实例，继承服务帐户的权限，并通过负载平衡器服务公开它:

```
apiVersion: v1
kind: ServiceAccount
metadata:
  name: k8s-admin
---
kind: Role
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  name: k8s-admin-role
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
  name: k8s-admin-role-binding
subjects:
- kind: ServiceAccount
  name: k8s-admin
  apiGroup: ""
roleRef:
  kind: Role
  name: k8s-admin-role
  apiGroup: ""
---
apiVersion: v1
kind: Service
metadata:
  name: my-ssh-svc
  labels:
    app: ssh
spec:
  type: LoadBalancer
  ports:
  - port: 2222
  selector:
    app: ssh
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-ssh
  labels:
    app: ssh
spec:
  replicas: 1
  selector:
    matchLabels:
      app: ssh
  template:
    metadata:
      labels:
        app: ssh
    spec:
      serviceAccountName: k8s-admin
      containers:
      - name: ssh
        image: lscr.io/linuxserver/openssh-server:latest
        ports:
        - containerPort: 2222
        env:
        - name: PUID
          value: "1000"
        - name: PGID
          value: "1000"
        - name: TZ
          value: "Australia/Brisbane"
        - name: USER_NAME
          value: "admin"
        - name: USER_PASSWORD
          value: "Password01!"
        - name: PASSWORD_ACCESS
          value: "true"
        - name: SUDO_ACCESS
          value: "true" 
```

注意，为了方便起见，这个 SSH 服务器允许密码访问，示例 YAML 文件嵌入了一个不安全的示例密码，并允许 sudo 访问。更可靠的解决方案是使用密钥文件进行身份验证。参考 [Docker Hub 文档](https://hub.docker.com/r/linuxserver/openssh-server)中显示如何使用密钥文件进行认证的示例。

将上面的 YAML 保存到一个名为`ssh.yaml`的文件中，并使用命令应用它:

```
kubectl apply -f ssh.yaml 
```

然后，您可以使用以下命令找到负载平衡器服务的 IP 地址或主机名:

```
kubectl get service my-ssh-svc 
```

在我的本地 Kubernetes 集群上，该命令返回:

```
NAME         TYPE           CLUSTER-IP      EXTERNAL-IP      PORT(S)          AGE
my-ssh-svc   LoadBalancer   10.96.164.169   172.21.255.202   2222:31628/TCP   29m 
```

然后，您可以使用以下命令 SSH 到外部 IP 地址:

```
ssh admin@172.21.255.202 -p 2222 
```

然后，您可以在 Kubernetes 集群上的 pod 内进行交互式会话。

## 安装和配置 kubectl

要对集群做任何有用的事情，您需要下载`kubectl`并将其配置为从 pod 中访问集群。使用命令下载并安装`kubectl`:

```
curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl"
sudo install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl 
```

默认情况下，pod 在`/var/run/secrets/kubernetes.io/serviceaccount`下挂载了许多文件，让 pod 与主机集群进行交互。要配置`kubectl`使用这些文件，将以下文件保存到`~/.kube.config`:

```
apiVersion: v1
clusters:
- cluster:
    certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
    server: https://kubernetes.default
  name: localk8s
contexts:
- context:
    cluster: localk8s
    user: user
  name: localk8s
current-context: localk8s
kind: Config
preferences: {}
users:
- name: user
  user:
    tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token 
```

现在，您可以从 SSH 会话运行`kubectl`并与父集群交互，为集群管理提供一个方便而安全的环境。

## 构建定制的 OpenSSH Docker 映像

下载`kubectl`和复制配置文件非常容易，但是 Kubernetes pods 的短暂性意味着最终容器将被删除和重新创建，迫使您再次下载和配置`kubectl`。

一个更好的解决方案是将`kubectl`及其配置文件烘焙成一个定制的 Docker 映像。这确保了文件在容器第一次启动时是可用的。

将以下内容保存到名为`Dockerfile`的文件中:

```
FROM lscr.io/linuxserver/openssh-server:latest
RUN curl -LO "https://dl.k8s.io/release/$(curl -L -s https://dl.k8s.io/release/stable.txt)/bin/linux/amd64/kubectl" && \
    install -o root -g root -m 0755 kubectl /usr/local/bin/kubectl
RUN printf 'apiVersion: v1\n\
clusters:\n\
- cluster:\n\
    certificate-authority: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt\n\
    server: https://kubernetes.default\n\
  name: localk8s\n\
contexts:\n\
- context:\n\
    cluster: localk8s\n\
    user: user\n\
  name: localk8s\n\
current-context: localk8s\n\
kind: Config\n\
preferences: {}\n\
users:\n\
- name: user\n\
  user:\n\
    tokenFile: /var/run/secrets/kubernetes.io/serviceaccount/token' >> /opt/kubeconfig
RUN printf 'mkdir /config/.kube \n\
cp /opt/kubeconfig /config/.kube/config' >> /etc/cont-init.d/100-kubeconfig 
```

然后，使用以下命令构建定制的 Docker 映像，其中的`yourdockerregistry`被替换为 Docker 注册表的名称，您可以将映像推送到该注册表中:

```
docker build . -t yourdockerregistry/openssh-server:latest 
```

将 Kubernetes YAML 文件中的`image`属性替换为:

```
image: yourdockerregistry/openssh-server:latest 
```

在使用您的定制映像创建了新的 SSH 服务器 pods 之后，`kubectl`及其配置文件就可以使用了，不需要先下载它们。

## 结论

在 Kubernetes 集群上运行 OpenSSH 的 bastion 主机为管理和调试任务提供了一个安全的入口点。通过定制 Docker 映像来包含像`kubectl`这样的公共工具，DevOps 团队可以依赖 bastion 主机来完成公共管理任务。

愉快的部署！