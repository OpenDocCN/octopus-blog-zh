# Kubernetes 资源的批量删除- Octopus 部署

> 原文：<https://octopus.com/blog/deleting-kubernetes-resources>

Kubernetes 使一次创建许多资源变得容易，用`kubectl apply -f filename.yaml`命令在一个复合 YAML 文件中创建所有资源。但是，如何删除多个资源而不单独指定它们呢？

在本文中，我将向您展示如何批量删除 Kubernetes 资源。

## Kubernetes 部署示例

让我们来看一个描述 Kubernetes 部署和服务的典型 YAML 文件:

```
apiVersion: v1
kind: Service
metadata:
  name: my-nginx-svc
  labels:
    app: nginx
spec:
  type: LoadBalancer
  ports:
  - port: 80
  selector:
    app: nginx
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-nginx
  labels:
    app: nginx
spec:
  replicas: 3
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.14.2
        ports:
        - containerPort: 80 
```

当这个 YAML 保存到一个名为`nginx.yaml`的文件中时，使用以下命令创建资源:

```
kubectl apply -f nginx.yaml 
```

然后，您可以查看使用以下命令创建的新资源:

```
kubectl get pods
kubectl get deployments
kubectl get services 
```

您会看到创建了三个 pod、一个部署和一个服务。pods 不是在 YAML 文件中直接定义的，而是由部署创建的，由于`replicas`属性被设置为`3`，所以创建了三个 pods。

## 从文件中删除资源

删除这些资源最简单的方法是使用`delete`命令并传递最初创建资源时使用的相同文件:

```
kubectl delete -f nginx.yaml 
```

如果您重新运行上面的`kubectl get`命令，您会看到 pod、部署和服务被删除。因为窗格由部署管理，所以删除部署也会删除窗格。

## 手动删除资源

为了手动删除特定类型的资源，`kubectl delete`命令接受一个定义要删除的资源类型的`--all`参数。例如，以下命令删除所有服务:

```
kubectl delete --all services 
```

您可以使用以下命令确认服务已删除:

```
kubectl get services 
```

此命令删除所有窗格:

```
kubectl delete --all pods 
```

该命令的输出如下所示:

```
pod "my-nginx-6595874d85-88jlr" deleted
pod "my-nginx-6595874d85-9w52c" deleted
pod "my-nginx-6595874d85-dpzds" deleted 
```

然而，当您确认 pod 被删除时，有趣的事情发生了。运行以下命令以列出任何窗格:

```
kubectl get pods 
```

请注意，仍然有 3 个 pod，输出如下所示:

```
NAME                        READY   STATUS    RESTARTS   AGE
my-nginx-6595874d85-2j4g8   1/1     Running   0          76s
my-nginx-6595874d85-4vrfb   1/1     Running   0          76s
my-nginx-6595874d85-4wj9p   1/1     Running   0          76s 
```

如果仔细观察，`kubectl get pods`命令显示的 pod 名称与`kubectl delete --all pods`命令返回的名称不同。这是因为 pod 由部署管理，当部署发现它管理的 pod 已被删除时，它会重新创建新的 pod 来完成其`replica`计数。

删除由部署管理的 pod 实质上是重新创建它们，如果您想要强制 pod 重新启动，这是很有用的。但是永久删除 pod 的唯一方法是删除它们的父部署。这是通过以下命令完成的:

```
kubectl delete --all deployments 
```

删除部署后，没有部署或窗格。

## 删除命名空间

命名空间是对相关资源进行分组的一种便捷方式。使用以下命令创建一个名为`foo`的新名称空间:

```
kubectl create namespace foo 
```

然后使用以下命令在新的名称空间中创建 NGINX 资源:

```
kubectl apply -f nginx.yaml -n foo 
```

使用命令列出资源:

```
kubectl get pods -n foo
kubectl get deployments -n foo
kubectl get services -n foo 
```

然后使用以下命令删除该命名空间:

```
kubectl delete namespace foo 
```

这会导致命名空间以及其中包含的所有资源被删除。

## 速记“所有”资源

当调用`kubectl`来引用 Kubernetes 资源类型的公共子集时，可以为资源类型传递`all`。因此，以下命令将删除服务、部署和 pod:

```
kubectl delete all --all 
```

`all`类型包括:

*   豆荚
*   服务
*   达蒙塞特
*   部署
*   复制集
*   状态集
*   工作
*   克朗乔布斯

## 删除与标签匹配的资源

标签用于丰富资源，元数据通常描述资源的用途、环境和版本等内容。您可以根据这些标签选择资源并删除它们。这使您可以有选择地删除资源组。

以下命令删除标签名为`app`设置为`nginx`的部署:

```
kubectl delete deployments -l app=nginx 
```

同样，您可以删除带有相同标签的服务:

```
kubectl delete service -l app=nginx 
```

## 试运行

批量删除资源很方便，但是很危险。幸运的是，`kubectl`有`--dry-run`参数，可以让您看到将要删除的资源，但不会实际删除它们。以下命令预览与`all`资源类型匹配的资源:

```
kubectl delete all --all --dry-run 
```

## 结论

使用`kubectl`批量删除资源很容易，在这篇文章中，你学习了如何删除资源:

*   在 YAML 文件中定义
*   匹配单一资源类型
*   分组在`all`资源类型中
*   包含在命名空间中
*   带有匹配的标签

您还学习了如何使用`--dry-run`参数来预览任何将被删除的资源。

愉快的部署！