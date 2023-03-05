# 通过 XML 模板或 CLI 脚本配置 wild fly-Octopus Deploy

> 原文：<https://octopus.com/blog/wildfly-cli-vs-xml>

当用像 Puppet、Chef、Ansible 等自动化工具构建 WildFly 服务器时，你将面临如何从库存下载到定制服务器的问题。定制的很大一部分是在构建过程中如何编辑配置文件(`standalone.xml`、`domain.xml`、`host.xml`和`host-slave.xml`)。

编辑这些 XML 文件时，您有两个主要选择:直接编辑文本，或者使用 WildFly CLI 进行更改。两者各有利弊，我们将在这篇博文中探讨。

## 直接编辑 XML

直接编辑 XML 配置文件可能是最自然的选择。由于 XML 文件只是纯文本，并且每个部署工具都有某种形式的模板支持，因此在基本模板中进行所需的更改并在部署期间复制定制版本是非常容易的。

这种方法有许多好处。

首先，您不需要 WildFly 的运行实例来进行配置更改。这消除了您在使用 CLI 工具时会遇到的一些麻烦，默认情况下，CLI 工具需要运行一个实例，并会强制您重新启动服务器以使一些更改生效。

第二，所有的更改本质上都是以“批处理”方式完成的，也就是说所有的更改都是一次性完成的。当您复制一个模板 XML 文件时，这听起来可能很明显，但是它确实删除了在通过 CLI 进行更改时必须考虑的一些内容。

但是，直接编辑 XML 配置文件有一些明显的缺点。

WildFly 使用的一些 XML 文件不是静态的。例如，WildFly 将使用当前部署的应用程序的详细信息更新`domain.xml`和`standalone.xml`文件。任何部署脚本的目标都是幂等的，但是如果您在每次部署时盲目地将一个新的 XML 配置文件复制到服务器上，您会发现自己丢失了胡乱做出的运行时更改。此外，您可能会发现带有新 XML 配置文件的服务器没有部署，这可能会给生产带来巨大的冲击。

如果没有预先考虑，您可能还会发现自己处于这样一种情况，即很难看出对 XML 文件做了什么更改。这些配置文件相当长，如果没有 diff 工具，几乎不可能发现定制模板中的更改。

出于同样的原因，在 WildFly 的新版本中，将您的定制应用到配置文件也是一个挑战。鉴于 WildFly 大约每年都会发布一个主要版本(从 WildFly 12 开始，这个发布时间表将会加快)，您真的希望能够轻松地将您的定制应用到下一个版本，即使只是为了利用 WildFly 后续版本中的安全补丁。

对于那些直接对 XML 配置文件应用更改的人，我的建议是为每个更改添加清晰的注释。例如，您可以使用如下代码更改配置以绑定到任何网络适配器:

```
<interfaces>
        <interface name="management">
            <!-- CHANGE: Bind to any IP address -->
            <any-address/>
        </interface>
        <interface name="public">
            <!-- CHANGE: Bind to any IP address -->
            <any-address/>
        </interface>
</interfaces> 
```

WildFly 在启动时会丢弃这些注释(WildFly 在运行时覆盖 XML 文件是 WildFly 运行时无法编辑这些文件的原因)，但通过简单的搜索字符串，您可以找到您在模板中所做的所有更改。这使得将更改移植到新版本变得容易，或者简单地理解您的定制版本与股票下载有何不同。

## 使用 CLI 更新

使用 WildFly CLI 工具来应用定制比编辑 XML 文件更高级，但它确实有许多优点。

您可能会发现，您的 CLI 脚本将适用于 WildFly 的新版本，无需任何更改。例如，以下 CLI 命令将公共接口绑定到任何地址:

```
/interface=public/:undefine-attribute(name=inet-address)
/interface=public/:write-attribute(name=any-address,value=true) 
```

我希望这些命令能在 WildFly 的所有最新版本中工作，也希望它们能在即将发布的版本中工作。这使得升级你的 WildFly 版本更加容易。

这些 CLI 命令还提供了一种简洁的方式来理解对股票下载所做的更改。与模板 XML 文件不同，在模板 XML 文件中，更改分散在一个大文件中，CLI 命令易于查看和理解。

因为您正在对当前配置应用有针对性的更改(而不是覆盖整个 XML 文件)，CLI 命令有可能是等幂的。通过使用 CLI 流控制语句，可以仅在当前设置不是所需状态时应用更改。

CLI 命令提供了一定程度的错误检查。公平地说，WildFly 中的 XML 验证非常严格，所以不太可能有无效的 XML 和可启动的服务器，但是当您试图做错什么时，CLI 会提供更直接的反馈。

对于复杂的环境，您可以利用 CLI Java 库，用任何 JVM 语言编写您的更改。Groovy 是一个不错的选择。使用 [Grape](http://docs.groovy-lang.org/latest/html/documentation/grape.html) 你可以删除任何需要的依赖项，你的脚本可以像任何其他可执行文件一样在 Linux 或 MacOS 环境中运行。

这个示例 groovy 脚本使用 WildFly CLI 库来执行所需的 CLI 命令。虽然这是一个非常简单的例子，但是可以修改它以适应更复杂的需求。

```
#!/usr/bin/env groovy

@Grab(group='org.wildfly.core', module='wildfly-embedded', version='2.2.1.Final')
@Grab(group='org.wildfly.security', module='wildfly-security-manager', version='1.1.2.Final')
@Grab(group='org.wildfly.core', module='wildfly-cli', version='3.0.0.Beta23')
import org.jboss.as.cli.scriptsupport.*

final DEFAULT_HOST = "localhost"
final DEFAULT_PORT = "9990"
final DEFAULT_PROTOCOL = "remote+http"

def cli = new CliBuilder()

cli.with {
    h longOpt: 'help', 'Show usage information'
    c longOpt: 'controller', args: 1, argName: 'controller', 'WildFly controller'
    d longOpt: 'port', args: 1, argName: 'port', type: Number.class, 'Wildfly management port'
    e longOpt: 'protocol', args: 1, argName: 'protocol', 'Wildfly management protocol i.e. remote+https'
    u longOpt: 'user', args: 1, argName: 'username', required: true, 'WildFly management username'
    p longOpt: 'password', args: 1, argName: 'password', required: true, 'WildFly management password'
}

def options = cli.parse(args)

if (!options) {
    return
}

if (options.h) {
    cli.usage()
    return
}

def jbossCli = CLI.newInstance()

jbossCli.connect(
                options.protocol ?: DEFAULT_PROTOCOL,
                options.controller ?: DEFAULT_HOST,
                Integer.parseInt(options.port ?: DEFAULT_PORT),
                options.user,
                options.password.toCharArray())

jbossCli.cmd("/:take-snapshot")
jbossCli.cmd("/interface=public/:undefine-attribute(name=inet-address)")
jbossCli.cmd("/interface=public/:write-attribute(name=any-address,value=true)")

System.exit(0) 
```

该脚本可以在以下情况下运行:

```
./script.groovy -u admin -p password01 
```

不过，运行 CLI 脚本也有缺点。

通常，您需要有一个正在运行的 WildFly 实例，以便运行您的 CLI 脚本。当服务器没有正确配置以便引导，从而可以对其运行 CLI 脚本时，这可能会很棘手。例如，接口绑定可能不正确，端口可能不正确，或者从属实例可能无法连接到域主节点。

这种情况可以通过运行[嵌入式服务器](http://www.mastertheboss.com/jbossas/wildfly9/configuring-wildfly-9-from-the-cli-in-offline-mode)来缓解，这是一种“离线”模式，让您无需运行服务器就可以访问 CLI。

您还需要知道哪些更改需要重新启动服务器，以及哪些更改需要作为批处理的一部分进行。这是在复制 XML 文件和启动服务器时不需要担心的两件事。

## 最后的想法

如果需要等幂，CLI 脚本是最佳选择。它们允许您只修改不处于所需状态的设置，并且是非破坏性的。CLI 脚本往往更容易移植到新的 WildFly 版本，并提供了您的实例如何不同于股票下载的简明定义。

对于不可变的环境，编辑 XML 文件的简单性是无法超越的。每个系统管理员都知道如何编辑 XML 文件(而对于新手来说，CLI 可能比较棘手)，每个部署工具都支持某种类型的模板，可以处理 XML 文件。只要您小心翼翼地清楚识别对股票配置文件所做的更改，将这些更改移植到 WildFly 的新版本是非常容易的。

如果您对 Java 应用程序的自动化部署感兴趣，[下载 Octopus Deploy](https://octopus.com/downloads) 的试用版，并查看一下[我们的文档](https://octopus.com/docs/deployments/java/deploying-java-applications)。