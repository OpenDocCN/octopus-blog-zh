# 构建 Raspberry Pi 集群的经验教训——Octopus 部署

> 原文：<https://octopus.com/blog/raspberry-pi-cluster-lessons-learned>

[![Lessons learned building a Raspberry Pi cluster](img/8ab7a9f277bb6b91a982b311e71129e3.png)](#)

像许多其他人一样，在新冠肺炎危机期间，我不得不呆在家里，所以我决定构建一个 Raspberry PI 集群，用 Docker Swarm 探索在其上运行应用程序。在这篇文章中，我分享了我在使用 Docker Swarm 的集群中运行新的 Raspberry Pi 4 时学到的一些经验。

## 为什么

我是一名开发人员，我对启动一个 Raspberry PI 项目感兴趣有一段时间了。我也对集群计算感兴趣，但是从来没有一个创建集群的好理由。后来，我读到有人在使用 Docker Swarm 的集群中运行新的 Raspberry Pi 4 取得了巨大成功，所以我认为这是了解更多信息的大好时机。

我带着几个目标开始了这个项目:

*   玩得开心，因为树莓皮是非常酷的硬件。
*   将一些计算资源从运行虚拟机的家庭虚拟机管理程序服务器转移到运行容器的小型节能 Raspberry PI 集群。
*   了解更多关于 Docker Swarm 的信息。

## Raspberry Pi 硬件

这些是我在项目中使用的部件:

我选择 C4Labs Cloudlet 案例，因为它允许我访问单个 Pi，而不必像可堆叠的案例那样拆卸整个案例。Cloudlet 机箱的原始设计是由 Pi 驱动风扇。如果我需要移除一个 Pi，我不想担心断开电缆，所以我用以下部件为风扇供电:

Cloudlet 案例为这些额外的部分提供了足够的空间。

最后，我需要一个网络交换机来连接 Pi 设备，但是我有一个备用的 8 端口，所以我不需要再买一个。Raspberry Pi 4 带有内置的无线适配器，所以你也可以利用它。

## 创建集群

有大量的博客文章向你展示了如何用 Docker Swarm 创建一个 Raspberry Pi 集群。我发现[如何用 Docker Swarm](https://howchoo.com/g/njy4zdm3mwy/how-to-run-a-raspberry-pi-cluster-with-docker-swarm) 运行一个 Raspberry Pi 集群很清楚，也很容易理解。你需要的所有细节都在那个帖子里。

## 操作蜂群

通过工作，我已经有了一些使用 Docker 和 Kubernetes (K8s)的经验，但是，我从来没有运行过 Docker Swarm。与独立的 Docker 实例相比，在 Docker 群体模式下运行时需要学习一些新概念。例如，在您通常使用`docker run`语句的地方，您可以在 swarm 中创建一个服务来使用`docker service create`运行容器。折腾了一段时间后，我有了一个完全可操作的虫群:

[![](img/39e178ef397b5345f9332bd083512d18.png)](#)

## 第 1 课:不是所有的容器都在 ARM 上运行

我做这个项目的初衷是通过运行容器来减轻我的虚拟机管理程序的负担。在搜索 [Docker Hub](https://hub.docker.com) 时，我发现很少有官方镜像会运行在 ARM 架构上。我通常可以找到某人制作的运行在 Raspberry Pi 上的图像，但它们通常是旧的，而不是当前的版本。

在某些情况下，您可以使用一种变通方法，告诉 Docker 不要将图像解析为架构:

*   `docker service create --name <somename> --no-resolve-image <image>`
*   `docker stack deploy --name <somename> --compose-file <somefile> --resolve-image never`

## 第二课:堆栈

栈是 Docker 群使用 [docker-compose](https://docs.docker.com/compose/) 的方式。事实上，它使用一个合成文件作为它的参数之一。就像 docker-compose 一样，您可以在一个文件中定义多个容器。默认情况下，[堆栈部署](https://docs.docker.com/engine/reference/commandline/stack_deploy/)会为堆栈创建一个网络，这样它们就可以互相通信。这使得仅通过名称就可以很容易地将一个容器引用到另一个容器。例如，下面是在我的 docker swarm 上运行[家庭助手](https://www.home-assistant.io/)的合成文件。这个堆栈由一个 MQTT 服务容器、一个家庭助理容器和一个 MySQL 数据库服务器容器组成。出于我将在后面介绍的原因，我将 Home Assistant 配置为对记录器使用 MySQL 后端:

```
version: "3.3"

services:

  mqtt:
    image: eclipse-mosquitto
    networks:
      - hass
    ports:
      - 1883:1883
    volumes:
      - /mnt/clusterpi/mqtt/db:/db
      - /etc/localtime:/etc/localtime:ro

  home-assistant:
    image: homeassistant/home-assistant
    networks:
      - hass
    ports:
      - 8123:8123
    volumes:
      - /mnt/clusterpi/home-assistant:/config
      - /etc/localtime:/etc/localtime:ro

  mysqldb:
    image: hypriot/rpi-mysql
    networks:
      - hass
    ports:
      - 3350:3306
    environment:
      MYSQL_ROOT_PASSWORD: "MySuperSecretPassword"
      MYSQL_DATABASE: "homeassistant"
      MYSQL_USER: "hassio"
      MYSQL_PASSWORD: "AnotherSuperSecretPassword"
    volumes:
      - /mnt/clusterpi/mysql:/var/lib/mysql

networks:
  hass: 
```

在 Home Assistant 的`configuration.yaml`文件中，你可以看到我在哪里创建了到 mysqldb 容器的连接:

```
 # Configure a default setup of Home Assistant (frontend, api, etc)
default_config:

# Uncomment this if you are using SSL/TLS, running in Docker container, etc.
# http:
#   base_url: example.duckdns.org:8123

# Text to speech
tts:
  - platform: google_translate

group: !include groups.yaml
automation: !include automations.yaml
script: !include scripts.yaml
scene: !include scenes.yaml
recorder:
  db_url: mysql://root:MySuperSecretPassword@mysqldb/homeassistant?charset=utf8 
```

## 第三课:群体成员行动一致

当与 docker 群组一起工作时，群组的所有成员都像一个整体一样行动。看上面的图，我们可以看到所有的容器都分布在群体的成员中。

当通过公开的端口连接到容器时，没有必要引用当前托管容器的节点。我们可以看到，`clusterpi-1`托管了名为`viz`的容器(这是图形的网页，`visualizer`)。`viz`容器端口被映射到主机端口 80，因此我可以通过 http://clusterpi-1 访问`visualizer`容器网页。也可以通过 http://clusterpi-4 访问它，即使`clusterpi-4`不是容器的当前主机。这也意味着任何被映射的端口都不能被另一个容器使用。`mysql_dev`容器是唯一可以映射到 3306 的东西，所有其他 MySQL 容器(除非它们是`mysql_dev`的复制品)都必须使用不同的端口。

## 第 4 课:持续的存储灾难

您从容器中学到的第一个教训是，当容器被销毁时，数据不会持久。为了防止这种情况，您需要配置容器，将它通常存储在内部的内容映射到外部卷。在 docker 主机上创建卷相当容易。使用`-v`开关:`-v /my/local/folder:/var/lib/data`指定本地文件夹到包含文件夹的映射。然而，这在 docker swarm 中出现了一个问题，因为无法保证每次运行时相同的主机都会运行相同的容器(好吧，从技术上来说，你可以通过约束来实现，但是这违背了 swarm 的目的)。为了解决这个问题，我们需要一个群体的所有成员都可以访问的东西，比如一个网络位置。

### 网络存储选项

有两种方法可以实现这一点；网络文件系统(NFS)或通用互联网文件系统(CIFS)。

NFS 通常用于 Linux/Unix 系统，CIFS 通常用于 Windows 系统，尽管 Windows 确实有 NFS 实现。在我的家庭环境中，我有一个用于文件服务器的 Windows 服务器，所以我选择使用 CIFS。

### 配置 CIFS

这篇文章很好地解释了如何在 Linux 上使用 CIFS，包括如何挂载 Windows 共享。我使用 CIFS 和`/etc/fstab`中的一个条目将我的 Pi 设备连接到我的文件服务器，尽管我确实有一个副作用。当我重新启动集群时，它们都可以访问文件共享，但是我的所有容器都在 Manager 节点上结束。从我的一个工人的`/etc/fstab`中删除条目证实了这确实是导致问题的原因。

#### 等待网络连接

所有容器都在 Manager 节点上结束的问题困扰了我一段时间。最终，我通过在`/etc/fstab`的 CIFS 线中添加另一个选项解决了这个问题；`_netdev`。`_netdev`开关延迟安装，直到联网被启用。在将它添加到所有 Pi 设备之后，管理器再次能够分发容器。

#### 绕过权限检查

有些容器在启动时会改变文件系统和文件系统权限，比如 Postgres。当完全在容器或本地文件系统挂载中执行时，这通常不是问题。然而，当通过 CIFS 使用 Windows 共享时，这通常会使容器崩溃，表明该操作不受支持。为了绕过这一点，我为 CIFS 找到了`noperm`选项，它跳过了权限检查，直接向前移动。这使得我可以通过 Postgres 解决这个问题，但是我从来没有能够获得一个 Windows CIFS 共享来使用 Postgres 容器(但这并不意味着这是不可能的)。

#### CIFS 和斯格利特不是朋友

为我的 HomeAssistant 容器成功配置了一个 CIFS 共享后，我发现它默认使用的 SqlLite 数据库有问题，它一直报告数据库被锁定。在遇到 pgAdmin (Postgres management)容器的同样问题后，我稍微搜索了一下，发现许多帖子描述了同样的行为，几乎所有的帖子都推荐 SqlLite 以外的东西。这就是我在 HomeAssistant 堆栈部署中包含记录器服务的 MySql 容器的原因。

## 结论

这对我来说是一个非常有趣的项目，我学到了很多关于 Docker Swarm 的知识。创建一个 Raspberry Pi 集群对于学习 Docker Swarm 和 Kubernetes 这样的概念非常有用。

[![](img/c7517a6e063243956abcd7f4b4c4c312.png)](#)