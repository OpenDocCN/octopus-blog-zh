# 在容器中运行 Octopus Deploy-Octopus Deploy

> 原文：<https://octopus.com/blog/octopuscontainer>

[![Octopus Docker Container Banner](img/fb2831595404d05adaf1510c82db6137.png)](#)

在当今快节奏的世界中，我们中的一些人真正需要的是一种更快的下载和运行 Octopus Deploy 的方法。谢天谢地，Octopus Deploy 容器现在已经可以使用了，它允许用户直接从 Docker 容器内部运行 Octopus 服务器或触手。[Octopus deploy/Octopus deploy](https://hub.docker.com/r/octopusdeploy/octopusdeploy/)和[Octopus deploy/触手](https://hub.docker.com/r/octopusdeploy/tentacle/)映像是在我们的标准 Octopus 服务器和触手构建过程中构建的，因此您可以始终确保有最新版本可用(升级过程非常简单*，您很快就会看到)。*

 *## 在容器中运行 Octopus 服务器

假设您手边有一个 SQL 数据库(我们将很快演示它本身甚至不是必需的),以最简单的方式启动 Octopus 服务器容器:

```
docker run -i --env sqlDbConnectionString=<MyConnectionString> octopusdeploy/octopusdeploy 
```

在这篇文章中，我不打算深入剖析 Docker 命令，而是鼓励你查看 [Docker 文档](https://docs.docker.com/engine/reference/run/)以获得更详细的解释。

作为优秀的容器拥护者，我们都知道应该把容器视为不可变的，并且可以在任何时候被拆除。为了解决这个问题，我们应该确保实例使用的服务器文件被写入更持久的内容。Octopus 服务器容器提供了几个挂载点，使维护这些文件变得更加容易。此外，尽管容器在端口`81`上公开了 Octopus Web 门户，但我们可能希望将其映射到主机上的特定端口。总的来说，上面的命令很好也很简单，但是提供我们自己的管理员凭证来访问新创建的实例可能比依赖缺省值更理想。

让我们为这个命令添加一些配置:

```
docker run --interactive --detach `
    --name OctopusServer `
    --publish "8081:80" `
    --volume "E:/Octopus/Repository:C:/Repository" `
    --volume "E:/Octopus/Artifacts:C:/Artifacts" `
    --volume "E:/Octopus/TaskLogs:C:/TaskLogs" `
    --env sqlDbConnectionString="Server=172.23.192.1,1433;Initial Catalog=Octopus;Persist Security Info=False;User ID=sa;Password=P@ssw0rdz;MultipleActiveResultSets=False;Connection Timeout=30;" `
    --env OctopusAdminUsername=Mario `
    --env OctopusAdminPassword=ItsAMe! `
    octopusdeploy/octopusdeploy:2018.4.0 
```

现在一切都已经从容器中具体化了，当我们期待已久的热门新 Octopus 特性可用时，我们可以轻松地升级 Octopus 服务器实例。首先，我们需要从最初的 Octopus 实例中获取主密钥。作为该容器启动过程的一部分，主密钥被写入日志，因此可以通过运行`docker logs OctopusServer`找到它。有了密钥，我们可以停止原来的容器，并使用新的主密钥环境变量和新的版本标记重新运行上面的运行命令:

```
docker run --interactive --detach `
    ...
    ...
    --env masterKey=7dnak8asdn23hjasd== `
     octopusdeploy/octopusdeploy:2018.4.1 
```

嘣，新版本将下载，执行任何必要的数据库迁移，并开始运行。*告诉你这是疯狂的简单。*查看 Octopus 服务器映像上的[我们的文档](https://octopus.com/docs/installation/octopus-in-container/octopus-server-container)以了解关于可用配置的更多详细信息。

## 在容器中运行触手

Octopus 触手容器也是可用的，但是由于 Octopus 中任何针对触手运行的部署任务都将在容器本身中运行，所以您不太可能使用它来更新 IIS 网站或部署 NodeJS 应用程序。这将开始提供更多的价值，当触手可执行程序将能够运行目前局限于“在服务器上运行”的任务。

当一个触手容器启动时，它用提供的服务器细节注册自己。随着未来的变化，触手也将在关闭时重新注册自己，但是该功能仅在最近的 Windows 容器版本中可用，因此在当前版本中尚未考虑 side。

```
docker run --interactive --detach  `
    --name MyTentacle `
    --env ServerApiKey=API-48AC758FF8912B `
    --env ServerUrl=http://myoctopus.acme.com `
    --env TargetEnvironment=Development `
    --env TargetRole=InnerContainer `
    octopusdeploy/octopusdeploy:2018.4.1 
```

触手可以配置为轮询或监听模式，同样，查看[我们的文档](https://octopus.com/docs/installation/octopus-in-container/octopus-tentacle-container)了解更多关于可用设置的详细信息。

## 没有 SQL？别担心

“可是抢！”我听到你说，“我没有一个 SQL 服务器可以运行 Octopus。”嗯，这很好，因为我们可以利用 [Docker Compose](https://docs.docker.com/compose/overview/) 在我们的 Octopus 服务器旁边构建一个 SQL 数据库容器。(*注意:围绕在容器中运行数据库以用于生产目的，有很多观点。我们倾向于同意。接下来的这些例子可能最好留给测试和实验。*

使用下面的`docker-compose.yml`文件:

```
version: '2.1'
services:
  db:
    image: microsoft/mssql-server-windows-express
    environment:
      sa_password: "${SA_PASSWORD}"
      ACCEPT_EULA: "Y"
    healthcheck:
      test: [ "CMD", "sqlcmd", "-U", "sa", "-P", "${SA_PASSWORD}", "-Q", "select 1" ]
      interval: 10s
      retries: 10
  octopus:
    image: octopusdeploy/octopusdeploy:${OCTOPUS_VERSION}
    environment:
      OctopusAdminUsername: "${OCTOPUS_ADMIN_USERNAME}"
      OctopusAdminPassword: "${OCTOPUS_ADMIN_PASSWORD}"
      sqlDbConnectionString: "Server=db,1433;Initial Catalog=Octopus;Persist Security Info=False;User ID=sa;Password=${SA_PASSWORD};MultipleActiveResultSets=False;Connection Timeout=30;"
    ports:
     - "8081:81"
    depends_on:
      db:
        condition: service_healthy
    stdin_open: true
    volumes:
      - "E:/Octopus/Repository:C:/Repository"
      - "E:/Octopus/TaskLogs:C:/TaskLogs"
networks:
  default:
    external:
      name: nat 
```

以及附带的`.env`文件:

```
SA_PASSWORD=P@ssw0rd!
OCTOPUS_VERSION=2018.3.13
OCTOPUS_ADMIN_USERNAME=admin
OCTOPUS_ADMIN_PASSWORD=SecreTP@ass 
```

只需运行:

```
docker-compose --project-name Octopus up -d 
```

在短暂的等待之后，您将拥有一个自包含的 SQL Server 和 Octopus Server 实例，可以为您的应用程序执行部署了。

在[我们的文档](https://octopus.com/docs/installation/octopus-in-container/docker-compose#octopus-server-and-tentacle)中，我们还指出了如何利用 Octopus 服务器映像上的`C:\Import`卷挂载来提供初始化数据以预填充数据库，然后在同一个`docker-compose`脚本中包含多个触角。

## 章鱼和容器

这种在容器内运行 Octopus 的最新能力进一步巩固了 Octopus Deploy 的承诺，即为希望在其环境中更好地利用容器的用户提供改进的解决方案。目前正在开发中的 Kubernetes 特性将允许在您的部署中与容器进行更深入的集成，而与平台无关。让我们知道您对我们未来方向的想法，看看我们的[现有的](https://octopus.com/docs/deployments/docker) Docker 产品和快乐(集装箱化)部署！*