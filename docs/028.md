# 用于基础设施测试的 arquillian-Octopus 部署

> 原文：<https://octopus.com/blog/arquillian-for-infrastructure-testing>

在之前的一篇博客文章中，我们看到了 Arquillian 如何解决测试真实应用服务器中真实对象的问题。虽然这可能是更传统的阿奎利亚语使用方式，但不是唯一的方式。

编写像 Octopus 这样的工具的挑战之一是它必须支持大量的 Java 应用服务器。目前，我们支持:

*   野花 10
*   野火 11
*   红帽 JBoss EAP 6
*   红帽 JBoss EAP 7
*   Tomcat 7
*   Tomcat 8
*   Tomcat 9

不仅如此，我们还支持独立模式或域模式下的 WildFly 和 JBoss EAP。这给了我们 10 多种不同的应用服务器配置来测试我们的代码。

Octopus 还支持将 JAR 和 WAR 文件等 Java 工件直接部署到文件系统中，这意味着 WebSphere、WebLogic、GlassFish、Payara 等应用服务器也可以集成到 Octopus 部署流程中。

我们本来可以用各种应用服务器创建虚拟机并运行测试，但是 Arquillian 能够运行真正的应用服务器并将这些服务器集成到单元测试中，这意味着我们可以用 JUnit 这样的标准单元测试库测试我们的代码，并直接从 Maven 运行这些测试。

在这篇博文中，我们将看看如何使用 Arquillian 进行基础设施测试。

## Maven POM 文件

配置 Arquillian 所需的大量工作都放在 Maven POM 文件中。Arquillian 已经有了全面的[入门文档](http://arquillian.org/guides/getting_started/)，所以我将在这里重点介绍一下。

我们将使用 Java 8 进行测试，因此我们相应地设置了`maven.compiler.source`和`maven.compiler.target`属性。

```
<properties>
    <maven.compiler.source>1.8</maven.compiler.source>
    <maven.compiler.target>1.8</maven.compiler.target>
</properties> 
```

我们将使用 Arquillian 物料清单(BOM)来提供我们稍后将定义的 Arquillian 依赖项的版本。

```
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.jboss.arquillian</groupId>
            <artifactId>arquillian-bom</artifactId>
            <version>1.1.14.Final</version>
            <scope>import</scope>
            <type>pom</type>
        </dependency>
    </dependencies>
</dependencyManagement> 
```

然后我们需要 Arquillian、JUnit 和 Apache HTTP 客户端依赖项。

注意我们使用的是 [Arquillian 变色龙](https://github.com/arquillian/arquillian-container-chameleon)，它提供了一种简单的方法来管理我们将要测试的各种容器(即应用服务器)。

Apache HTTP 客户机将用于执行一个简单的测试，以确保应用服务器正在运行。

```
<dependencies>
   <dependency>
       <groupId>org.apache.httpcomponents</groupId>
       <artifactId>httpclient</artifactId>
       <version>4.5.3</version>
   </dependency>
   <dependency>
       <groupId>org.jboss.arquillian.junit</groupId>
       <artifactId>arquillian-junit-container</artifactId>
       <scope>test</scope>
   </dependency>
   <dependency>
       <groupId>junit</groupId>
       <artifactId>junit</artifactId>
       <version>4.12</version>
       <scope>test</scope>
   </dependency>
   <dependency>
       <groupId>org.arquillian.container</groupId>
       <artifactId>arquillian-container-chameleon</artifactId>
       <version>1.0.0.Final-SNAPSHOT</version>
       <scope>test</scope>
   </dependency>
</dependencies> 
```

我们将定义两个 Maven 概要文件:`Tomcat8`和`WildFly9`。这些配置文件定义了`arquillian.launch`系统属性，该属性用于选择 Chameleon 将为我们下载的应用服务器，并设置将要运行的测试的路径。在`WildFly9`概要文件中，我们将运行`wildfly9`容器，并在`src/test/java/wildfly9`目录中运行测试。

```
<profile>
    <id>WildFly9</id>
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-surefire-plugin</artifactId>
                <configuration>
                    <systemPropertyVariables>
                        <arquillian.launch>wildfly9</arquillian.launch>
                    </systemPropertyVariables>
                    <includes>
                        <include>**/wildfly9/**</include>
                    </includes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</profile> 
```

`Tomcat8`概要文件将运行`tomcat8`容器，并从`src/test/java/tomcat8`目录运行测试。

```
<profile>
  <id>Tomcat8</id>
  <build>
      <plugins>
          <plugin>
              <groupId>org.apache.maven.plugins</groupId>
              <artifactId>maven-surefire-plugin</artifactId>
              <configuration>
                  <systemPropertyVariables>
                      <arquillian.launch>tomcat8</arquillian.launch>
                  </systemPropertyVariables>
                  <includes>
                      <include>**/tomcat8/**</include>
                  </includes>
              </configuration>
          </plugin>
      </plugins>
  </build>
</profile> 
```

我们使用概要文件是因为一次只能运行一个应用服务器。概要文件允许我们选择用 Maven 命令行参数测试的服务器类型。我们还分割了测试类文件，以确保只有给定服务器的测试才能使用所选择的概要文件运行。

这是完整的 POM 文件。

```
<?xml version="1.0" encoding="UTF-8"?>
<project  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>com.octopus</groupId>
    <artifactId>arquillian-infrastructure-testing</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <maven.compiler.source>1.8</maven.compiler.source>
        <maven.compiler.target>1.8</maven.compiler.target>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>org.jboss.arquillian</groupId>
                <artifactId>arquillian-bom</artifactId>
                <version>1.1.14.Final</version>
                <scope>import</scope>
                <type>pom</type>
            </dependency>
        </dependencies>
    </dependencyManagement>

    <dependencies>
        <dependency>
            <groupId>org.apache.httpcomponents</groupId>
            <artifactId>httpclient</artifactId>
            <version>4.5.3</version>
        </dependency>
        <dependency>
            <groupId>org.jboss.arquillian.junit</groupId>
            <artifactId>arquillian-junit-container</artifactId>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>junit</groupId>
            <artifactId>junit</artifactId>
            <version>4.12</version>
            <scope>test</scope>
        </dependency>
        <dependency>
            <groupId>org.arquillian.container</groupId>
            <artifactId>arquillian-container-chameleon</artifactId>
            <version>1.0.0.Final-SNAPSHOT</version>
            <scope>test</scope>
        </dependency>
    </dependencies>

    <profiles>
        <profile>
            <id>WildFly9</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration>
                            <systemPropertyVariables>
                                <arquillian.launch>wildfly9</arquillian.launch>
                            </systemPropertyVariables>
                            <includes>
                                <include>**/wildfly9/**</include>
                            </includes>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
        <profile>
            <id>Tomcat8</id>
            <build>
                <plugins>
                    <plugin>
                        <groupId>org.apache.maven.plugins</groupId>
                        <artifactId>maven-surefire-plugin</artifactId>
                        <configuration>
                            <systemPropertyVariables>
                                <arquillian.launch>tomcat8</arquillian.launch>
                            </systemPropertyVariables>
                            <includes>
                                <include>**/tomcat8/**</include>
                            </includes>
                        </configuration>
                    </plugin>
                </plugins>
            </build>
        </profile>
    </profiles>
</project> 
```

## 配置阿奎利亚语

Arquillian 需要一个名为`arquillian.xml`的文件，该文件定义了它可以运行的容器(或应用服务器)。在我们的例子中，我们需要定义两个容器来匹配 Maven 概要文件分配给`arquillian.launch`系统属性的值。

### Tomcat 8 容器

`tomcat8`容器从发布给 Maven 的[二进制发行版下载 Tomcat 8.0.47。](http://central.maven.org/maven2/org/apache/tomcat/tomcat/8.0.47/)

我们还需要定义 Arquillian 用来访问 Tomcat 提供的管理器应用程序的用户名和密码。在这种情况下，我们对两者都使用`arquillian`。

为了实际定义用户`arquillian`，我们用一个名为`tomcat8-server.xml`的定制配置文件来配置 Tomcat。

```
<container qualifier="tomcat8">
    <configuration>
        <property name="target">tomcat:8.0.47:managed</property>
        <!-- relative to CATALINA_BASE/conf; catalinaBase is set by chameleon itself -->
        <property name="serverConfig">../../../../../src/test/resources/tomcat8-server.xml</property>
        <property name="user">arquillian</property>
        <property name="pass">arquillian</property>
    </configuration>
</container> 
```

在`tomcat8-server.xml`文件中，我们引用了`tomcat-users.xml`文件。

```
<!-- Global JNDI resources
     Documentation at /docs/jndi-resources-howto.html
-->
<GlobalNamingResources>
    <!-- Editable user database that can also be used by
         UserDatabaseRealm to authenticate users
    -->
    <Resource name="UserDatabase" auth="Container"
              type="org.apache.catalina.UserDatabase"
              description="User database that can be updated and saved"
              factory="org.apache.catalina.users.MemoryUserDatabaseFactory"
              pathname="${catalina.home}/../../../../src/test/resources/tomcat-users.xml"/>
</GlobalNamingResources> 
```

最后在`tomcat-users.xml`文件中，我们定义了用户`arquillian`。

```
<?xml version='1.0' encoding='utf-8'?>
<tomcat-users>
    <user username="arquillian" password="arquillian" roles="manager-script"/>
</tomcat-users> 
```

CATALINE_BASE 和 CATALINA_HOME(被`tomcat-server.xml`文件中的`${catalina.home}`引用和`arquillian.xml`文件中`serverConfig`设置的相对位置)为`target/server/tomcat_8.0.47/apache-tomcat-8.0.47`，是 Arquillian Chameleon 下载 Tomcat 的位置。父目录引用的长字符串从这个目录返回到 test `resources`目录。

### WildFly 9 容器

WildFly 容器没有那么复杂。它也疯狂地下载了发布给 Maven 的[二进制发行版。唯一的其他设置是服务器配置文件的名称，我们将其设置为`standalone.xml`。](http://central.maven.org/maven2/org/wildfly/wildfly-dist/9.0.0.Final/)

```
<container qualifier="wildfly9">
    <configuration>
        <property name="target">wildfly:9.0.0.Final:managed</property>
        <property name="serverConfig">standalone.xml</property>
    </configuration>
</container> 
```

### 完整的 Arquillian 配置文件

这是完整的`arquillian.xml`文件。

```
<arquillian  xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
            xsi:schemaLocation="
        http://jboss.org/schema/arquillian
        http://jboss.org/schema/arquillian/arquillian_1_0.xsd">

    <container qualifier="tomcat8">
        <configuration>
            <property name="target">tomcat:8.0.47:managed</property>
            <!-- relative to CATALINA_BASE/conf; catalinaBase is set by chameleon itself -->
            <property name="serverConfig">../../../../../src/test/resources/tomcat8-server.xml</property>
            <property name="user">arquillian</property>
            <property name="pass">arquillian</property>
        </configuration>
    </container>

    <container qualifier="wildfly9">
        <configuration>
            <property name="target">wildfly:9.0.0.Final:managed</property>
            <property name="serverConfig">standalone.xml</property>
        </configuration>
    </container>

</arquillian> 
```

## 配置阿奎利亚变色龙

Chameleon 通常不需要太多的配置，尽管我们在这里使用的版本(1.0.0.Final-SNAPSHOT)必须提供一个定制的`containers.yaml`文件来配置 Tomcat 8。这个版本基于来自[变色龙 GitHub repo](https://github.com/arquillian/arquillian-container-chameleon/blob/58f36a281721d8ce8776fa60275d322e7c3b9b24/arquillian-chameleon-container-model/src/main/resources/chameleon/default/containers.yaml) 的版本。

Chameleon 在`containers.yaml`文件上提供了[文档](https://github.com/arquillian/arquillian-container-chameleon#development)，所以我在这里就不赘述了。

```
- name: JBoss EAP
  versionExpression: 7.*
  adapters:
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-remote:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-managed:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: &EAP7_CONFIG
        jbossHome: ${dist}
    - type: embedded
      gav: org.wildfly.arquillian:wildfly-arquillian-container-embedded:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: *EAP7_CONFIG
  dist: &EAP7_DIST
    gav: org.jboss.as:jboss-as-dist:zip:${version}
  exclude: &EAP7_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
- name: JBoss EAP Domain
  versionExpression: 7.*
  adapters:
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-managed:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *EAP7_CONFIG
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-remote:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: &EAP7_DIST
    gav: org.jboss.as:jboss-as-dist:zip:${version}
  exclude: &EAP7_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
- name: JBoss EAP
  versionExpression: 6.0.*
  adapters:
    - type: remote
      gav: org.jboss.as:jboss-as-arquillian-container-remote:7.1.2.Final
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.jboss.as:jboss-as-arquillian-container-managed:7.1.2.Final
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: &EAP_CONFIG
        jbossHome: ${dist}
    - type: embedded
      gav: org.jboss.as:jboss-as-arquillian-container-embedded:7.1.2.Final
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: *EAP_CONFIG
  dist: &EAP_DIST
    gav: org.jboss.as:jboss-as-dist:zip:${version}
  exclude: &EAP_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
    - "*:wildfly-arquillian-protocol-jmx"
    - "*:jboss-as-arquillian-testenricher-msc"
    - "*:jboss-as-arquillian-protocol-jmx"
- name: JBoss EAP Domain
  versionExpression: 6.0.*
  adapters:
    - type: managed
      gav: org.jboss.as:jboss-as-arquillian-container-domain-managed:7.1.2.Final
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *EAP_CONFIG
    - type: remote
      gav: org.jboss.as:jboss-as-arquillian-container-domain-remote:7.1.2.Final
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: &EAP_DIST
    gav: org.jboss.as:jboss-as-dist:zip:${version}
  exclude: &EAP_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
    - "*:wildfly-arquillian-protocol-jmx"
    - "*:jboss-as-arquillian-testenricher-msc"
    - "*:jboss-as-arquillian-protocol-jmx"
- name: JBoss EAP
  versionExpression: 6.*
  adapters:
    - type: remote
      gav: org.jboss.as:jboss-as-arquillian-container-remote:7.1.3.Final
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.jboss.as:jboss-as-arquillian-container-managed:7.1.3.Final
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: *EAP_CONFIG
    - type: embedded
      gav: org.jboss.as:jboss-as-arquillian-container-embedded:7.1.3.Final
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: *EAP_CONFIG
  dist: *EAP_DIST
  exclude: *EAP_EXCLUDE
- name: JBoss EAP Domain
  versionExpression: 6.*
  adapters:
    - type: managed
      gav: org.jboss.as:jboss-as-arquillian-container-domain-managed:7.1.3.Final
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *EAP_CONFIG
    - type: remote
      gav: org.jboss.as:jboss-as-arquillian-container-domain-remote:7.1.3.Final
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: *EAP_DIST
  exclude: *EAP_EXCLUDE
- name: JBoss AS
  versionExpression: 7\.0\.[0-2]\.(.*)$|7\.1\.[0-1]\.(.*)$
  adapters:
    - type: remote
      gav: org.jboss.as:jboss-as-arquillian-container-remote:${version}
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.jboss.as:jboss-as-arquillian-container-managed:${version}
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: &AS_CONFIG
        jbossHome: ${dist}
    - type: embedded
      gav: org.jboss.as:jboss-as-arquillian-container-embedded:${version}
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: *AS_CONFIG
  dist: &AS_DIST
    gav: org.jboss.as:jboss-as-dist:zip:${version}
  exclude: &AS_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
    - "*:wildfly-arquillian-protocol-jmx"
    - "*:jboss-as-arquillian-testenricher-msc"
    - "*:jboss-as-arquillian-protocol-jmx"
- name: JBoss AS Domain
  versionExpression: 7\.0\.[0-2]\.(.*)$|7\.1\.[0-1]\.(.*)$
  adapters:
    - type: managed
      gav: org.jboss.as:jboss-as-arquillian-container-domain-managed:${version}
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *AS_CONFIG
    - type: remote
      gav: org.jboss.as:jboss-as-arquillian-container-domain-remote:${version}
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: &AS_DIST
    gav: org.jboss.as:jboss-as-dist:zip:${version}
  exclude: &AS_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
    - "*:wildfly-arquillian-protocol-jmx"
    - "*:jboss-as-arquillian-testenricher-msc"
    - "*:jboss-as-arquillian-protocol-jmx"
- name: WildFly
  versionExpression: 8.*
  adapters:
    - type: remote
      gav: org.wildfly:wildfly-arquillian-container-remote:${version}
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.wildfly:wildfly-arquillian-container-managed:${version}
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: &WF_CONFIG
        jbossHome: ${dist}
    - type: embedded
      gav: org.wildfly:wildfly-arquillian-container-embedded:${version}
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: &WF_EMBEDD_CONFIG
        jbossHome: ${dist}
        modulePath: ${dist}/modules
      dependencies:
        - org.jboss.remotingjmx:remoting-jmx:2.0.1.Final
        - org.jboss.logging:jboss-logging:3.2.1.Final
  dist: &WF_DIST
    gav: org.wildfly:wildfly-dist:zip:${version}
  exclude: &WF_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
    - "*:wildfly-arquillian-protocol-jmx"
    - "*:jboss-as-arquillian-testenricher-msc"
    - "*:jboss-as-arquillian-protocol-jmx"
- name: WildFly Domain
  versionExpression: 8.*
  adapters:
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-managed:${version}
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *WF_CONFIG
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-remote:${version}
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: &WF_DIST
    gav: org.wildfly:wildfly-dist:zip:${version}
  exclude: &WF_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
    - "*:wildfly-arquillian-protocol-jmx"
    - "*:jboss-as-arquillian-testenricher-msc"
    - "*:jboss-as-arquillian-protocol-jmx"
- name: WildFly
  versionExpression: 9.*
  adapters:
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-remote:1.1.0.Final
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-managed:1.1.0.Final
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: *WF_CONFIG
    - type: embedded
      gav: org.wildfly.arquillian:wildfly-arquillian-container-embedded:1.1.0.Final
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: *WF_EMBEDD_CONFIG
      dependencies:
        - org.jboss.remotingjmx:remoting-jmx:2.0.1.Final
  dist: *WF_DIST
  exclude: &WF9_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
- name: WildFly Domain
  versionExpression: 9.*
  adapters:
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-managed:1.1.0.Final
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *WF_CONFIG
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-remote:1.1.0.Final
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: *WF_DIST
  exclude: &WF9_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
- name: WildFly
  versionExpression: 10.*
  adapters:
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-remote:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.remote.RemoteDeployableContainer
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-managed:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.managed.ManagedDeployableContainer
      configuration: *WF_CONFIG
    - type: embedded
      gav: org.wildfly.arquillian:wildfly-arquillian-container-embedded:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.embedded.EmbeddedDeployableContainer
      configuration: *WF_EMBEDD_CONFIG
  dist: *WF_DIST
  exclude: &WF10_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
- name: WildFly Domain
  versionExpression: 10.*
  adapters:
    - type: managed
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-managed:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.domain.managed.ManagedDomainDeployableContainer
      configuration: *WF_CONFIG
    - type: remote
      gav: org.wildfly.arquillian:wildfly-arquillian-container-domain-remote:2.0.1.Final
      adapterClass: org.jboss.as.arquillian.container.domain.remote.RemoteDomainDeployableContainer
  dist: *WF_DIST
  exclude: &WF10_EXCLUDE
    - org.jboss.arquillian.test:*
    - org.jboss.arquillian.testenricher:*
    - org.jboss.arquillian.container:*
    - org.jboss.arquillian.core:*
    - org.jboss.arquillian.config:*
    - org.jboss.arquillian.protocol:*
    - org.jboss.shrinkwrap.api:*
    - org.jboss.shrinkwrap:*
    - org.jboss.shrinkwrap.descriptors:*
    - org.jboss.shrinkwrap.resolver:*
    - "*:wildfly-arquillian-testenricher-msc"
# Older versions of Glassfish (before 3.1.2) are no longer supported
- name: GlassFish
  versionExpression: ^3\.1\.[2-9]{1}(\.[0-9])*$
  adapters:
    - &GF_REMOTE
          type: remote
          gav: org.jboss.arquillian.container:arquillian-glassfish-remote-3.1:1.0.1
          adapterClass: org.jboss.arquillian.container.glassfish.remote_3_1.GlassFishRestDeployableContainer
    - &GF_MANAGED
      type: managed
      gav: org.jboss.arquillian.container:arquillian-glassfish-managed-3.1:1.0.1
      adapterClass: org.jboss.arquillian.container.glassfish.managed_3_1.GlassFishManagedDeployableContainer
      configuration:
        glassFishHome: ${dist}
        outputToConsole: true
    - &GF_EMBEDDED
      type: embedded
      gav: org.jboss.arquillian.container:arquillian-glassfish-embedded-3.1:1.0.1
      adapterClass: org.jboss.arquillian.container.glassfish.embedded_3_1.GlassFishContainer
      requireDist: false
      dependencies:
        - org.glassfish.main.extras:glassfish-embedded-all:${version}
  dist: &GF_DIST
    gav: org.glassfish.main.distributions:glassfish:zip:${version}

- name: GlassFish
  versionExpression: 4.*
  adapters:
    - *GF_REMOTE
    - *GF_MANAGED
    - *GF_EMBEDDED
  dist: *GF_DIST

- name: Payara
  versionExpression: 4.*
  adapters:
    - *GF_REMOTE
    - *GF_MANAGED
    - type: embedded
      gav: org.jboss.arquillian.container:arquillian-glassfish-embedded-3.1:1.0.1
      adapterClass: org.jboss.arquillian.container.glassfish.embedded_3_1.GlassFishContainer
      requireDist: false
      dependencies:
        - fish.payara.extras:payara-embedded-all:${version}
  dist:
    gav: fish.payara.distributions:payara:zip:${version}

- name: Tomcat
  versionExpression: 6.*
  adapters:
    - type: remote
      gav: org.jboss.arquillian.container:arquillian-tomcat-remote-6:1.0.0.CR9
      adapterClass: org.jboss.arquillian.container.tomcat.remote.Tomcat6RemoteContainer
    - type: managed
      gav: org.jboss.arquillian.container:arquillian-tomcat-managed-6:1.0.0.CR9
      adapterClass: org.jboss.arquillian.container.tomcat.managed.Tomcat6ManagedContainer
      configuration: &TOMCAT_MANAGED_CONFIG
        catalinaHome: ${dist}
        catalinaBase: ${dist}
  dist:
    gav: http://archive.apache.org/dist/tomcat/tomcat-6/v${version}/bin/apache-tomcat-${version}.zip

- name: Tomcat
  versionExpression: 7.*
  adapters:
    - type: remote
      gav: org.jboss.arquillian.container:arquillian-tomcat-remote-7:1.0.0.CR9
      adapterClass: org.jboss.arquillian.container.tomcat.remote.Tomcat7RemoteContainer
    - type: managed
      gav: org.jboss.arquillian.container:arquillian-tomcat-managed-7:1.0.0.CR9
      adapterClass: org.jboss.arquillian.container.tomcat.managed.Tomcat7ManagedContainer
      configuration: *TOMCAT_MANAGED_CONFIG
  dist: &TOMCAT_DIST
    gav: org.apache.tomcat:tomcat:zip:${version}

- name: Tomcat
  versionExpression: 8.0.*
  adapters:
    - type: remote
      gav: org.jboss.arquillian.container:arquillian-tomcat-remote-8:1.0.0.CR9
      adapterClass: org.jboss.arquillian.container.tomcat.remote.Tomcat8RemoteContainer
    - type: managed
      gav: org.jboss.arquillian.container:arquillian-tomcat-managed-8:1.0.0.CR9
      adapterClass: org.jboss.arquillian.container.tomcat.managed.Tomcat8ManagedContainer
      configuration: *TOMCAT_MANAGED_CONFIG
  dist: *TOMCAT_DIST 
```

当你读到这篇文章的时候，Chameleon 可能已经发布了一个已经配置了 Tomcat 的版本。

## 运行测试

为了简单起见，我们对 Tomcat 和 WildFly 运行的测试将是对它们的根目录的 HTTP GET 请求。通常，在这些测试中，您会进行 API 调用、部署应用程序或您的应用程序需要针对这些应用服务器做的任何事情。

这些测试的格式与[上一篇博文](/blog/arquillian-testing)中的测试格式非常相似。但是有两个很大的不同。

首先，我们没有`@Deployment`方法，因为我们没有在应用服务器内部运行任何代码。

第二，我们的测试方法有`@RunAsClient`注释，这意味着这段代码在应用服务器之外运行。我们认为这些测试是应用服务器的客户端，而不是测试部署到它们上面的代码。通过从外向内看，我们可以测试我们的代码，就好像它是一个与 Arquillian 管理的应用服务器一起工作的外部应用程序。

```
package tomcat8;

import org.apache.http.client.methods.CloseableHttpResponse;
import org.apache.http.client.methods.HttpGet;
import org.apache.http.impl.client.CloseableHttpClient;
import org.apache.http.impl.client.HttpClients;
import org.jboss.arquillian.container.test.api.RunAsClient;
import org.jboss.arquillian.junit.Arquillian;
import org.junit.Assert;
import org.junit.Test;
import org.junit.runner.RunWith;

import java.io.IOException;

@RunWith(Arquillian.class)
public class TomcatTest {
    @Test
    @RunAsClient
    public void connectToTomcat() throws IOException {
        try (CloseableHttpClient httpclient = HttpClients.createDefault()) {
            final HttpGet httpGet = new HttpGet("http://localhost:8080");
            try (CloseableHttpResponse response1 = httpclient.execute(httpGet)) {
                final int responseCode = response1.getStatusLine().getStatusCode();
                Assert.assertTrue(responseCode >= 200);
                Assert.assertTrue(responseCode <= 399);
            }
        }
    }
} 
```

## 运行测试

要针对特定的 Maven 概要文件运行测试，请提供`-P`参数，如下所示:

```
mvn clean verify -PTomcat8 
```

Arquillian 将为指定的应用服务器下载二进制发行版，启动服务器，并运行您的测试。这是输出的样子。

```
/Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/bin/java -Dmaven.multiModuleProjectDirectory=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting "-Dmaven.home=/Users/matthewcasperson/Library/Application Support/JetBrains/Toolbox/apps/IDEA-U/ch-0/172.4574.11/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3" "-Dclassworlds.conf=/Users/matthewcasperson/Library/Application Support/JetBrains/Toolbox/apps/IDEA-U/ch-0/172.4574.11/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3/bin/m2.conf" "-javaagent:/Users/matthewcasperson/Library/Application Support/JetBrains/Toolbox/apps/IDEA-U/ch-0/172.4574.11/IntelliJ IDEA.app/Contents/lib/idea_rt.jar=54994:/Users/matthewcasperson/Library/Application Support/JetBrains/Toolbox/apps/IDEA-U/ch-0/172.4574.11/IntelliJ IDEA.app/Contents/bin" -Dfile.encoding=UTF-8 -classpath "/Users/matthewcasperson/Library/Application Support/JetBrains/Toolbox/apps/IDEA-U/ch-0/172.4574.11/IntelliJ IDEA.app/Contents/plugins/maven/lib/maven3/boot/plexus-classworlds-2.5.2.jar" org.codehaus.classworlds.Launcher -Didea.version=2017.2.6 test -P Tomcat8
objc[12174]: Class JavaLaunchHelper is implemented in both /Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/bin/java (0x1057b94c0) and /Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/jre/lib/libinstrument.dylib (0x1077f54e0). One of the two will be used. Which one is undefined.
[INFO] Scanning for projects...
[WARNING]
[WARNING] Some problems were encountered while building the effective model for com.octopus:arquillian-infrastructure-testing:jar:1.0-SNAPSHOT
[WARNING] 'build.plugins.plugin.version' for org.apache.maven.plugins:maven-surefire-plugin is missing. @ line 75, column 29
[WARNING]
[WARNING] It is highly recommended to fix these problems because they threaten the stability of your build.
[WARNING]
[WARNING] For this reason, future Maven versions might no longer support building such malformed projects.
[WARNING]
[INFO]                                                                         
[INFO] ------------------------------------------------------------------------
[INFO] Building arquillian-infrastructure-testing 1.0-SNAPSHOT
[INFO] ------------------------------------------------------------------------
[INFO]
[INFO] --- maven-resources-plugin:2.6:resources (default-resources) @ arquillian-infrastructure-testing ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] skip non existing resourceDirectory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/src/main/resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:compile (default-compile) @ arquillian-infrastructure-testing ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-resources-plugin:2.6:testResources (default-testResources) @ arquillian-infrastructure-testing ---
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent!
[INFO] Copying 4 resources
[INFO]
[INFO] --- maven-compiler-plugin:3.1:testCompile (default-testCompile) @ arquillian-infrastructure-testing ---
[INFO] Nothing to compile - all classes are up to date
[INFO]
[INFO] --- maven-surefire-plugin:2.12.4:test (default-test) @ arquillian-infrastructure-testing ---
[INFO] Surefire report directory: /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/surefire-reports

-------------------------------------------------------
 T E S T S
-------------------------------------------------------
Running tomcat8.TomcatTest
Nov 28, 2017 6:59:24 PM org.jboss.arquillian.container.tomcat.managed.TomcatManagedContainer start
INFO: Starting Tomcat with: [/Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/jre/bin/java, -Djava.util.logging.config.file=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/conf/logging.properties, -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager, -Dcom.sun.management.jmxremote.port=8089, -Dcom.sun.management.jmxremote.ssl=false, -Dcom.sun.management.jmxremote.authenticate=false, -Xmx512m, -XX:MaxPermSize=128m, -classpath, /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/bin/bootstrap.jar:/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/bin/tomcat-juli.jar, -Djava.endorsed.dirs=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/endorsed, -Dcatalina.base=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47, -Dcatalina.home=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47, -Djava.io.tmpdir=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/temp, org.apache.catalina.startup.Bootstrap, -config, /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/conf/../../../../../src/test/resources/tomcat8-server.xml, start]
Java HotSpot(TM) 64-Bit Server VM warning: ignoring option MaxPermSize=128m; support was removed in 8.0
28-Nov-2017 18:59:24.858 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server version:        Apache Tomcat/8.0.47
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server built:          Sep 29 2017 13:46:41 UTC
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Server number:         8.0.47.0
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Name:               Mac OS X
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log OS Version:            10.13.1
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Architecture:          x86_64
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Java Home:             /Library/Java/JavaVirtualMachines/jdk1.8.0_151.jdk/Contents/Home/jre
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Version:           1.8.0_151-b12
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log JVM Vendor:            Oracle Corporation
28-Nov-2017 18:59:24.860 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_BASE:         /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47
28-Nov-2017 18:59:24.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log CATALINA_HOME:         /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47
28-Nov-2017 18:59:24.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.config.file=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/conf/logging.properties
28-Nov-2017 18:59:24.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.util.logging.manager=org.apache.juli.ClassLoaderLogManager
28-Nov-2017 18:59:24.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcom.sun.management.jmxremote.port=8089
28-Nov-2017 18:59:24.861 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcom.sun.management.jmxremote.ssl=false
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcom.sun.management.jmxremote.authenticate=false
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Xmx512m
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -XX:MaxPermSize=128m
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.endorsed.dirs=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/endorsed
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcatalina.base=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Dcatalina.home=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.startup.VersionLoggerListener.log Command line argument: -Djava.io.tmpdir=/Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/temp
28-Nov-2017 18:59:24.862 INFO [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent The APR based Apache Tomcat Native library which allows optimal performance in production environments was not found on the java.library.path: /Users/matthewcasperson/Library/Java/Extensions:/Library/Java/Extensions:/Network/Library/Java/Extensions:/System/Library/Java/Extensions:/usr/lib/java:.
28-Nov-2017 18:59:24.966 INFO [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["http-nio-8080"]
28-Nov-2017 18:59:24.985 INFO [main] org.apache.tomcat.util.net.NioSelectorPool.getSharedSelector Using a shared selector for servlet write/read
28-Nov-2017 18:59:24.987 INFO [main] org.apache.coyote.AbstractProtocol.init Initializing ProtocolHandler ["ajp-nio-8009"]
28-Nov-2017 18:59:24.989 INFO [main] org.apache.tomcat.util.net.NioSelectorPool.getSharedSelector Using a shared selector for servlet write/read
28-Nov-2017 18:59:24.989 INFO [main] org.apache.catalina.startup.Catalina.load Initialization processed in 386 ms
28-Nov-2017 18:59:25.010 INFO [main] org.apache.catalina.core.StandardService.startInternal Starting service Catalina
28-Nov-2017 18:59:25.010 INFO [main] org.apache.catalina.core.StandardEngine.startInternal Starting Servlet Engine: Apache Tomcat/8.0.47
28-Nov-2017 18:59:25.017 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deploying web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/docs
28-Nov-2017 18:59:25.306 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/docs has finished in 289 ms
28-Nov-2017 18:59:25.306 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deploying web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/manager
28-Nov-2017 18:59:25.336 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/manager has finished in 30 ms
28-Nov-2017 18:59:25.336 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deploying web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/examples
28-Nov-2017 18:59:25.535 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/examples has finished in 199 ms
28-Nov-2017 18:59:25.536 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deploying web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/ROOT
28-Nov-2017 18:59:25.552 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/ROOT has finished in 16 ms
28-Nov-2017 18:59:25.552 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deploying web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/host-manager
28-Nov-2017 18:59:25.564 INFO [localhost-startStop-1] org.apache.catalina.startup.HostConfig.deployDirectory Deployment of web application directory /Users/matthewcasperson/Development/ArquillianInfrastructureTesting/target/server/tomcat_8.0.47/apache-tomcat-8.0.47/webapps/host-manager has finished in 12 ms
28-Nov-2017 18:59:25.566 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["http-nio-8080"]
28-Nov-2017 18:59:25.571 INFO [main] org.apache.coyote.AbstractProtocol.start Starting ProtocolHandler ["ajp-nio-8009"]
28-Nov-2017 18:59:25.573 INFO [main] org.apache.catalina.startup.Catalina.start Server startup in 583 ms
Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 3.047 sec

Results :

Tests run: 1, Failures: 0, Errors: 0, Skipped: 0

[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time: 4.906 s
[INFO] Finished at: 2017-11-28T18:59:26+10:00
[INFO] Final Memory: 14M/309M
[INFO] ------------------------------------------------------------------------

Process finished with exit code 0 
```

## 结论

通过利用 Arquillian 来下载、初始化和清理它所支持的各种应用服务器，我们可以很容易地测试作为客户机与这些服务器交互的代码。这意味着只需花费一点点精力来设置 Maven 和配置 Arquillian 容器，就可以针对许多不同的应用服务器和这些服务器的许多不同版本运行测试。这种方法对我们来说非常有效，Java 代码充当了 Octopus 和 Java 应用服务器之间的粘合剂。

你可以从 [GitHub](https://github.com/OctopusDeploy/ArquillianInfrastructureTesting) 下载这篇博文的源代码。

如果您对 Java 应用程序的自动化部署感兴趣，[下载 Octopus Deploy](https://octopus.com/downloads) 的试用版，并查看一下[我们的文档](https://octopus.com/docs/deployments/java/deploying-java-applications)。