# 通过 minikube - Octopus Deploy 使用本地映像

> 原文：<https://octopus.com/blog/local-images-minikube>

minikube 为 DevOps 团队提供了一个本地开发 Kubernetes 集群。在本地开发 Kubernetes 应用程序通常需要构建和部署本地 Docker 映像。虽然 minikube 将下载托管在外部 Docker 注册表上的任何 Docker 映像，但公开本地构建的映像需要将映像加载到 minikube 集群中，并注意一些抛出无用错误消息的边缘情况。

在本文中，我将向您展示如何将本地构建的 Docker 映像部署到 minikube。

## 构建 Docker 图像

章鱼水下样本应用程序提供了一个简单的 Docker 图像用于测试。运行以下命令来克隆 git repo:

```
git clone https://github.com/OctopusSamples/octopus-underwater-app.git 
```

输入项目目录:

```
cd octopus-underwater-app 
```

然后使用以下命令构建映像:

```
docker build . -t underwater 
```

最后，使用以下命令运行 Docker 映像:

```
docker run -p 5000:80 underwater 
```

然后在`http://localhost:5000`可以获得示例 web 应用程序。

## 将图像推送到 minikube

使用以下命令将本地映像推送到 minikube 是一个简单的过程:

```
minikube image load underwater 
```

## 部署映像

要部署映像，请将以下 YAML 保存到名为`underwater.yaml`的文件中:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: underwater
  labels:
    app: web
spec:
  selector:
    matchLabels:
      app: web
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: underwater
          image: underwater
          imagePullPolicy: Never
          ports:
            - containerPort: 80 
```

然后使用以下命令部署应用程序:

```
kubectl apply -f underwater.yaml 
```

然后，应用程序被成功部署到 minikube 集群。

需要注意的是，Docker 图像没有标签，这意味着它有默认标签`latest`。因此，上面 YAML 中的`image`属性可以替换为以下文本，因为这两个图像引用是等效的:

```
image: underwater:latest 
```

使用带有`latest`标签的图像有特殊含义，需要将`imagePullPolicy`设置为`Never`(或`IfNotPresent`)。要了解原因，您需要了解默认的图像拉取策略。

## 使用最新图像

Kubernetes 文档提供了关于[默认图像拉取策略](https://kubernetes.io/docs/concepts/containers/images/#imagepullpolicy-defaulting)的建议:

> *   如果省略 imagePullPolicy 字段，并且容器图像的标记为:latest，imagePullPolicy 将自动设置为 Always
> *   如果省略 imagePullPolicy 字段，并且不指定容器图像的标记，imagePullPolicy 将自动设置为 Always。
> *   如果省略 imagePullPolicy 字段，并且为容器图像指定的标记不是:latest，则 imagePullPolicy 会自动设置为 IfNotPresent。

为了理解这一点的重要性，部署以下没有设置`imagePullPolicy`值的 YAML:

```
apiVersion: apps/v1
kind: Deployment
metadata:
  name: underwater
  labels:
    app: web
spec:
  selector:
    matchLabels:
      app: web
  replicas: 1
  strategy:
    type: RollingUpdate
  template:
    metadata:
      labels:
        app: web
    spec:
      containers:
        - name: underwater
          image: underwater
          ports:
            - containerPort: 80 
```

使用以下命令查看由此部署创建的单元的状态:

```
kubectl get pods 
```

您会看到状态为`ImagePullBackOff`:

```
NAME                          READY   STATUS             RESTARTS   AGE
underwater-847d6f9646-pvzxb   0/1     ImagePullBackOff   0          15m 
```

这是因为您部署了一个带有`latest`标签的图像，并且没有指定`imagePullPolicy`，这意味着使用了默认值`Always`。这反过来意味着 minikube 试图下载图片`docker.io/underwater:latest`，因为没有注册的图片默认为`docker.io`。图像`docker.io/underwater:latest`不存在，因此出现`ImagePullBackOff`错误。

有两种方法可以解决这个问题:

*   将`imagePullPolicy`设置为`Never`或`IfNotPresent`
*   为图像添加标签，例如`docker build . -t underwaterapp:0.0.1`和`minikube image load underwater:0.0.1`

## 结论

在 minikube 中使用本地构建的 Docker 映像是一个简单的过程，但是您需要了解有关映像拉取策略的规则，以确保 Kubernetes 不会试图从默认的 Docker 注册表中下载不存在的映像。

愉快的部署！