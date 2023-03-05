# 空心广口瓶介绍-章鱼部署

> 原文：<https://octopus.com/blog/hollow-jars>

我过去曾经写过关于应用服务器和 UberJAR 之间的区别。简而言之，应用服务器是一个并行托管多个 JavaEE 应用程序的环境，而 UberJAR 是一个自包含的可执行 JAR 文件，它启动并托管单个应用程序。

在这两种风格之间还有另一种风格的 JAR 文件，叫做空心 JAR。

## 什么是空心罐子？

空心 JAR 是一个单独的可执行 JAR 文件，像 UberJAR 一样，包含启动一个应用程序所需的代码。但是与 UberJAR 不同，空心 JAR 不包含应用程序代码。

然后，典型的中空 JAR 部署将由两个文件组成:中空 JAR 本身，以及保存应用程序代码的 WAR 文件。然后，空心 JAR 引用 WAR 文件来执行，从那时起，这两个文件就像 UberJAR 一样运行。

乍一看，用两个文件组成一个空心 JAR 部署，而不是用一个文件组成 UberJAR，这似乎没有什么效果，但是有一些好处。

主要的好处来自于空心 JAR 部署对的 JAR 组件不会经常改变。虽然您可能期望每天多次部署 WAR 半部署的新版本，但是 JAR 半部署将在几周或几个月内保持静态。这在构建分层容器图像时特别有用，因为只需要将修改后的 WAR 文件添加为容器图像层。

同样，您也可以使用 AWS 自动伸缩组这样的技术来减少构建时间。因为 JAR 文件不经常更改，所以可以将它放入 AMI 中，而 WAR 文件可以在 EC2 实例部署时通过 EC2 [用户数据](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/user-data.html)字段中的脚本下载。

## 建造一个中空的罐子

要想看到实际使用的空心罐子，让我们来看看如何用[野生蜂群](http://wildfly-swarm.io/)来制作一个。

对于这个演示，我们将构建运行一个 [Ticket Monster](https://github.com/jboss-developer/ticket-monster) 演示应用程序所需的一对空心 JAR 部署文件。Ticket Monster 是一个示例应用程序，创建它是为了演示一系列 JavaEE 技术，旨在构建一个在传统应用服务器上运行的 WAR 文件。

为了构建中空罐子的一半，我们将使用 [SwarmTool](https://wildfly-swarm.gitbooks.io/wildfly-swarm-users-guide/content/getting-started/tooling/swarmtool.html) 。与 WildFly Swarm 不同，它通常需要在 Maven 项目中进行特殊配置来构建 UberJAR 或空心 JAR，SwarmTool 通过检查现有的 WAR 文件并构建一个空心 JAR 来容纳它。这是一种将现有应用程序迁移到 Swarm 平台的简洁方式，无需修改现有的构建过程。

首先，从[https://github.com/jboss-developer/ticket-monster](https://github.com/jboss-developer/ticket-monster)克隆票怪源代码。我们感兴趣的代码在`demo`子文件夹下。

为了适应 Java 9 和 SwarmTool，我们需要对`demo`子文件夹下的`pom.xml`文件做两处修改。

首先，我们需要添加对`javax.xml.bind:jaxb-api`的依赖。这是因为`java.xml`包不再是 [Java 9](https://stackoverflow.com/a/43574427/157605) 的一部分。如果您尝试在 Java 9 下编译应用程序而没有这个额外的依赖项，您将收到以下错误:

```
java.lang.NoClassDefFoundError: javax/xml/bind/JAXBException 
```

下面的 XML 添加了所需的依赖项:

```
<dependencies>
  <dependency>
        <groupId>javax.xml.bind</groupId>
        <artifactId>jaxb-api</artifactId>
        <version>2.3.0</version>
    </dependency>
</dependencies> 
```

第二个变化是将 Ticket Monster 使用的 Jackson 库嵌入到 WAR 文件中。在原始源代码中，Jackson 库的作用域是`provided`，这意味着我们期望应用服务器(或者在我们的例子中是 Hollow JAR)提供这个库。

```
<dependency>
    <groupId>org.jboss.resteasy</groupId>
    <artifactId>resteasy-jackson-provider</artifactId>
    <scope>provided</scope>
</dependency> 
```

然而，我们将使用的 Swarm 版本与 Ticket Monster 应用程序使用的 Jackson 库版本不同。这种不匹配意味着 Swarm 提供的 Jackson 库版本无法识别 Ticket Monster 使用的`@JsonIgnoreProperties`注释，从而导致一些序列化错误。

幸运的是，所需要的只是使用默认的作用域，这将把 Jackson 库的正确版本嵌入到 WAR 文件中。嵌入在 WAR 文件中的依赖项优先，因此应用程序将按预期运行。

```
<dependency>
  <groupId>org.jboss.resteasy</groupId>
  <artifactId>resteasy-jackson-provider</artifactId>
</dependency> 
```

我们现在可以像构建任何其他 WAR 项目一样构建 Ticket Monster 应用程序。以下命令将构建 WAR 文件。

```
mvn package 
```

现在我们需要使用 SwarmTool 来构建空心罐子。在本地下载 SwarmTool JAR 文件。

```
wget https://repo1.maven.org/maven2/org/wildfly/swarm/swarmtool/2017.12.1/swarmtool-2017.12.1-standalone.jar 
```

然后造一个空心罐子。

```
java -jar swarmtool-2017.12.1-standalone.jar -d com.h2database:h2:1.4.196 --hollow target/ticket-monster.war 
```

`-d com.h2database:h2:1.4.196`参数指示 SwarmTool 将内存数据库依赖关系中的 H2 添加到空心 JAR 中。SwarmTool 可以通过扫描应用程序代码引用的类来检测启动 WAR 文件所需的大多数依赖项。然而，它不能像数据库驱动程序那样检测依赖性，所以我们需要手动告诉 SwarmTool 包含这种依赖性。

`--hollow`参数指示 SwarmTool 构建一个不嵌入 WAR 文件的空心 JAR。如果我们不使用这个参数，WAR 文件将被嵌入到生成的 JAR 文件中，创建一个 UberJAR 而不是一个空心的 JAR。

现在，我们有了两个文件，它们组成了我们的 Hollow JAR 部署。位于`target/ticket-monster.war`的 WAR 文件包含我们的应用程序，而`ticket-monster-swarm.jar`文件是我们的空心罐子。

## 执行空心罐子

要运行该应用程序，请使用以下命令。

```
java -jar ticket-monster-swarm.jar target/ticket-monster.war 
```

然后，您可以打开 http://localhost:8080 来查看该应用程序。

[![Ticket Monster](img/3c03e81fd9bd0f5d678cbe08bcf2c5af.png)](#)

## 结论

空心 JAR 是一种简洁的解决方案，它在保留 UberJAR 的便利性的同时，在部署策略上提供了很大的灵活性。你可以从博客文章[中找到更多关于不同策略的信息，关于肥胖、瘦、空心和优步](https://developers.redhat.com/blog/2017/08/24/the-skinny-on-fat-thin-hollow-and-uber/)。

如果您对 Java 应用程序的自动化部署感兴趣，[下载 Octopus Deploy](https://octopus.com/downloads) 的试用版，并查看我们的文档。