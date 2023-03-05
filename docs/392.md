# 检查 Kubernetes pod CPU 和内存- Octopus 部署

> 原文：<https://octopus.com/blog/kubernetes-pod-cpu-memory>

在 Linux 中，使用`top`或`htop`命令跟踪本地进程的资源使用相对容易。但是如何跟踪分布在 Kubernetes 集群中的 pod 的资源使用情况呢？

在本文中，我将向您展示如何在 Kubernetes 中查看 pods 的 CPU 和内存使用情况。

## 度量服务器

[metrics server](https://github.com/kubernetes-sigs/metrics-server) 为 Kubernetes 集群提供了一个轻量级和高度可伸缩的解决方案，用于收集 CPU 和内存资源。尽管度量服务器没有内置到 Kubernetes 中，但是大多数集群要么捆绑它，要么提供一个简单的解决方案来实现它。

安装度量服务后，将显示 pod 资源，命令如下:

```
kubectl top pod 
```

节点资源使用情况可通过以下命令获得:

```
kubectl top node 
```

以下错误表明没有安装度量服务器:

```
error: Metrics API not available 
```

在这种情况下，您可以使用这里的指令[安装 metrics server。](https://github.com/kubernetes-sigs/metrics-server)

## c 组资源使用

如果度量服务不可用，仍然可以通过进入交互式会话并打印 cgroup 接口文件的内容来确定单个 pod 的内存使用情况。

使用以下命令进入交互会话，将`podname`替换为您希望检测的 pod 的名称:

```
kubectl exec -it podname -- sh 
```

使用以下命令打印当前的内存使用情况:

```
cat /sys/fs/cgroup/memory/memory.usage_in_bytes 
```

使用以下命令打印当前的 cpu 使用情况:

```
cat /sys/fs/cgroup/cpu/cpuacct.usage 
```

注意`cpuacct.usage`返回的值并不是立即有用的，因为[返回的是](https://www.kernel.org/doc/Documentation/cgroup-v1/cpuacct.txt):

> 该组获得的 CPU 时间(纳秒)

将这个值转换成更有用的度量标准，如 CPU 使用率，需要一些计算。这篇关于栈交换的文章提供了更多的细节，这篇 T2 的 Python 代码提供了一个有用的实例。

您可以在 Linux 内核文档中找到关于这些文件[的更多信息。](https://www.kernel.org/doc/Documentation/cgroup-v1/00-INDEX)

## cgroup2 资源使用情况

如果目录`/sys/fs/cgroup/memory`或`/sys/fs/cgroup/cpu`不存在，您可能正在使用 cgroups v2 的系统上工作。

在装有 cgroups v2 的系统上，使用以下命令打印当前的内存使用情况:

```
cat /sys/fs/cgroup/memory.current 
```

使用以下命令打印当前的 cpu 使用情况:

```
cat /sys/fs/cgroup/cpu.stat 
```

这将打印一个名为`usage_usec`的文件。与 cgroup v1 `cpuacct.usage`文件返回的值一样，该值必须转换为 CPU 使用率百分比才能使用。

注意，`usage_usec`值是以毫秒为单位测量的，不像`cpuacct.usage`文件返回的值是以纳秒为单位。通过将`usage_usec`值乘以 1000 将其转换为纳秒，此时它可以用于由`cpuacct.usage`文件返回的相同计算中。

你可以在 Linux 内核文档中找到关于这些文件[的更多信息。](https://www.kernel.org/doc/Documentation/cgroup-v2.txt)

## 结论

metrics server 提供了一种方便的方法来检查 Kubernetes pods 和节点的 CPU 和内存资源。也可以通过检查 cgroup 接口文件来手动找到这些值，尽管需要一些手动计算来确定 CPU 使用率的百分比。

愉快的部署！