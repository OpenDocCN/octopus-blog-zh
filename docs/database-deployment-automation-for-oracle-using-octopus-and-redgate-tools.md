# 使用 Octopus 和 Redgate Deployment Suite for Oracle 实现数据库部署自动化

> 原文：<https://octopus.com/blog/database-deployment-automation-for-oracle-using-octopus-and-redgate-tools>

[![Database deployment automation using Octopus and Redgate Deployment Suite for Oracle](img/e439d64b809371741120ba5b7fb74bf3.png)](#)

在加入 Octopus Deploy 之前，我在一个. NET 应用程序上工作了大约三年，使用 Oracle 作为它的数据库。在 Octopus Deploy 版本发布前几年，我开始在那里工作。这些都是艰难的部署。一切都是手动的，我们只能在周六凌晨 2 点进行部署，部署需要两到四个小时。

谢天谢地，那些日子已经过去了。今天可用的工具比以前先进了好几光年。在这篇文章中，我将介绍如何将更改部署到 Oracle 数据库中。本文的目标是使用 TeamCity 作为构建服务器来构建整个 CI/CD 管道(尽管核心概念确实转移到了 Jenkins、Bamboo、TFS/VSTS/Azure DevOps)，Octopus Deploy 作为部署工具，Redgate Oracle 工具集在数据库端完成繁重的工作。

*免责声明:*我在 2010 年至 2013 年间使用甲骨文。Oracle 实例是 10g，我用 Benthic 和 SQL Developer 查询 Oracle。从那时起，发生了很多变化。我是一名 SQL Server 人员，毫无疑问，我在本文中做了一些愚蠢的事情，这些事情不再是最佳实践。

## 入门指南

如果您想继续学习，您需要下载并安装以下工具:

为了从 Oracle 下载任何东西，您必须创建一个帐户。我选择了个人版，因为就像 SQL Server Developer Edition 一样，它功能齐全，但在使用许可证时会受到限制，而且我只在这个演示中使用它。

出于本文的目的，我将部署到运行在 Windows 机器上的同一个 Oracle 数据库。然而，在现实环境中，您应该遵循与此类似的设置。

[![](img/3a58a984d90a5d42b193b8e74fb72d55.png)](#)

通过这种设置，您可以在 Octopus Deploy 和 Oracle 数据库前面的 VIP 之间的一个[工人](https://octopus.com/docs/infrastructure/workers)或一个 [jumpbox 运行触手](https://octopus.com/docs/infrastructure/windows-targets)上安装一个触手。

## 创建源数据库

Redgate 的 Oracle 工具集是一个基于状态的工具。对于那些没有读过我以前文章的人来说，基于状态的工具是一种将数据库的期望状态保存到源代码控制中的工具。在部署过程中，会生成一个独特的增量脚本来针对 Oracle 数据库运行。该增量脚本将仅用于该环境的数据库。下一个环境将获得一个新的增量脚本。

我要从头开始。我设置了一个虚拟机并安装了 Oracle。接下来，我将创建一个源数据库。这个数据库将代表我希望所有其他数据库处于的理想状态。我将设置我的数据库，然后使用 [Redgate 的 Oracle](https://www.red-gate.com/products/oracle-development/source-control-for-oracle/) 源代码控制将它签入源代码控制。

我使用 Oracle 提供的[数据库创建助手](https://docs.oracle.com/cd/B16254_01/doc/server.102/b14196/install003.htm)来创建数据库。

[![](img/d6f129dd9e75681c0fc93f610259f8a9.png)](#)

我将添加一个包含两列的新表:

[![](img/3d00ef342ec20267f4c5692c9b95893a.png)](#)

我还会加入一个序列。对于那些不熟悉 Oracle 的人来说，当您想对 ID 字段使用自动递增的数字时，需要一个序列。好吧，反正是在老派神谕里。看起来事情有点变化。

[![](img/a8a3279485a8078585f27b3dfccb043d.png)](#)

最后，我将把序列和 ID 字段绑定在一起。这为我提供了一些准备部署到空白数据库的模式更改。您的更改很可能会更加复杂。

[![](img/993f055862e68c94a7ccf0fd45684d04.png)](#)

## 将 Oracle 与源代码管理捆绑在一起

既然我们想要创建的表已经准备好了，是时候将表和序列定义放入源代码控制中了。为此，我使用 [Redgate 的 Oracle](https://www.red-gate.com/products/oracle-development/source-control-for-oracle/) 源代码控制。

当我们第一次启动应用程序时，我们看到一个选项，**创建一个新的源代码管理项目…** :

[![](img/cede94afd16a301d5755dc9211560190.png)](#)

首先，您需要配置数据库连接。不要忘记测试您的连接，这样您就知道它工作正常:

[![](img/03e6ba5d32df78a1644e18033dcb2a3c.png)](#)

数据库连接准备就绪。接下来，我们配置要使用的源代码控制系统。这个存储库将是一个 git 存储库，但是不使用内置的 git 功能。我更喜欢使用工作文件夹。这样我就可以使用我选择的 git GUI，并获得 git 的全部功能，特别是分支。一旦这个过程开始工作，下一步就是在每个开发人员的机器上安装 Oracle。使用专用实例，开发人员可以同时签入他们的源代码和数据库更改。而不是在共享服务器上进行数据库更改，然后等待代码被签入以利用该更改。

如您所见，我有一个空的工作文件夹:

[![](img/394e22895b9e29ebcaca60cf688fa114.png)](#)

我可以将该工作文件夹的目录输入到 Redgate 的 Oracle 源代码控制中:

[![](img/7d3b8546d50c8d453c25c5c34f32d90a.png)](#)

我需要选择我想要使用的模式。我将表放在 SourceDB 模式中，因此我将选择这个模式:

[![](img/7b1755f4eb0558faf320bdee3aebef9f.png)](#)

最棒的是，它会显示将要创建的所有文件夹，以及将要创建这些文件夹的目录:

[![](img/7c4edc6f68f8adc7b0c07d5945c300aa.png)](#)

该工具将监控该数据库和模式的任何更改。我将保持名称不变，以免以后混淆:

[![](img/173d2224a8d70ccc92ef5934fd2ca957.png)](#)

有四个要签入的变更:

[![](img/7b8c8fd52199f5a579fffcabcbee81cd.png)](#)

当我单击大箭头时，会出现一个摘要屏幕。对于那些使用过 Redgate 的 SQL 源代码控制的人来说，这应该很熟悉:

[![](img/dbf0edb1a6444d793404f35f5af5e8f2.png)](#)

单击“保存”按钮会显示所有内容都已成功保存:

[![](img/ea2f6a099ffe70b8bbb0d86095f07040.png)](#)

打开 Git 扩展，您可以看到已经创建的所有文件:

[![](img/9ea59861327cf26504424995017df096.png)](#)

我将提交这些更改。现在是时候建立一个构建并将这些更改打包成一个 zip 文件供 Octopus Deploy 使用了。

## 设置构建服务器

在我的 TeamCity 实例中，我创建了一个简单的项目来打包数据库，将包发布到 Octopus Deploy，并在 Octopus Deploy 中创建一个新版本。首先是包数据库。对于这个构建步骤，我将打包整个 db/src 文件夹(包括任何附加的模式)。现在它只包含了 *SourceDB* 模式:

[![](img/df59f3ac4eb52af42eb67c1d8e3012ca.png)](#)

推销产品应该非常简单:

[![](img/e134f145d05b92eef132aea0d44032a1.png)](#)

在 Octopus Deploy 中，我设置了一个非常简单的部署过程。此时的目标是确保所有东西都能成功打包、推送和部署。我还不太担心部署过程(这将在几分钟内完成):

[![](img/e53f00fd1413701ec0125b76ddfddfa0.png)](#)

回到 TeamCity，我将在我创建一个发布步骤中使用这个新项目:

[![](img/8d11956a4ed95cd1bb8eb7714376b104.png)](#)

现在是关键时刻，第一次运行构建。和...我搞砸了。当然，我做到了。没有什么事情第一次会完美无缺:

[![](img/1c78c3311b56b1936d335a7b3cbe56ba.png)](#)

试了几次，但我让构建工作正常了:

[![](img/34ef3817b8504998ecc19a816a5c183b.png)](#)

问题是我在打包路径的开头放了一个/应该是 db/src，就像这样:

[![](img/d16b10872e2f8ac8c85b89ac7d99b0c0.png)](#)

如果我从 Octopus 下载软件包并检查它，我可以看到所有创建的文件都在那里:

[![](img/4b395aec0d2ea6f37deac3ef2ab9ec87.png)](#)

我们有构建服务器打包、发布和触发部署。现在是时候去 Octopus 并完成这个过程了。

## 配置目标数据库

如果您像我一样第一次设置所有东西，那么您将需要配置一个目标数据库。我使用数据库创建助手配置了一个名为 **DestDB** 的新数据库。该数据库的用户名也是 **DestDB** 。

如您所见，我没有在这个数据库上设置任何东西:

[![](img/6aa40943efb5c04c30ac0422fe8cdb83.png)](#)

## 设置 Octopus 部署

如前面的截图所示，您应该在 Octopus Deploy 和 Oracle 数据库之间设置一个跳转框。这台机器需要安装 Redgate 的 Oracle Tool-belt、 **SQL*Plus** 和标准的 **tnsnames.ora** 文件。tnsnames.ora 文件需要包含您需要从这个跳转框连接到的所有主机(数据库服务器)。

由于 Redgate 的 Oracle Tool-belt 中的一个怪癖，您需要作为一个帐户而不是作为本地系统帐户运行触手；否则，您将得到错误消息，指出该工具未被激活，即使它已被激活。请按照[这些说明](https://octopus.com/docs/infrastructure/windows-targets/running-tentacle-under-a-specific-user-account)进行设置。

我已经向社区库添加了两个新的步骤模板。 [Redgate -创建 Oracle 版本](https://library.octopus.com/step-templates/0aa40fba-949c-4065-a438-010349c3fd0c/actiontemplate-redgate-create-oracle-release)和[运行 Oracle SQLPlus 脚本](https://library.octopus.com/step-templates/c7cd3ab4-5dfb-4f8d-957e-1940ed30359c/actiontemplate-run-oracle-sqlplus-script)。请下载并安装在您的 Octopus Deploy 实例上。

我整理的过程非常简单。第一步生成一个报告和一个增量脚本，在生产前和生产中，DBA 批准更改，然后对数据库运行增量脚本。

我故意让这一步只生成一个增量脚本和一个报告文件。通过包含`/deploy`命令行开关，可以让 Redgate 的 Oracle 工具为您完成部署。我省略了命令行开关，因为我觉得首先在过程中建立信任并让一个人批准更改是很重要的。社区图书馆是开源的。您可以随意复制并调整该步骤以满足您的需求:

[![](img/96cf9db1695be724da35f89a9fa0ece3.png)](#)

在第一步中，有几个选项需要完成。我已经尽力包含尽可能多的帮助文本。如果有不清楚的地方，请发电子邮件给 support@octopus.com，让我们知道，我们会解决的。

[![](img/55ab4a85838e626b693a1acb737374ab.png)](#)

在该步骤的最后，要求您提供源模式和目标模式。这是由于设置红门的工具。step-template 包装了命令行，并提供了模式名作为选项，因此 step 模板也必须提供它作为选项:

[![](img/7f578dc85d24f13b983d900abfbf60b3.png)](#)

第二步模板将采用任何脚本，并使用 SQL*Plus 针对 Oracle 数据库运行它。这一步只需要运行脚本的路径和访问 Oracle 数据库所需的凭证。

**请注意**，Redgate - Create Oracle 发布步骤将在导出目录中生成一个名为 **Delta.sql** 的文件。我想让这个脚本尽可能通用，这就是为什么您必须提供完整的路径:

[![](img/5b7f8b4bc32c4f5d773a6dbe7fc4a2d2.png)](#)

我喜欢 Oracle 工具的一点是它生成的报告显示了包中存储的脚本和目标数据库之间的差异。Redgate - Create Oracle 版本将使其成为一个工件，供您下载和查看:

[![](img/263bb1dd311a3605bb8cc17404c2d414.png)](#)

此外，它还使 delta 脚本成为一个可以下载的工件:

[![](img/a74cadec405a186308cdd0032a134917.png)](#)

## 运行首次部署

Octopus 现在已经配置完毕，可以开始第一次测试了。我只打算部署到 dev。

[![](img/6bc49c8f3e588f690e593874b6f57697.png)](#)

让我们检查一下数据库来确定一下。是的，一切都在那里。

[![](img/d8bb1cac15f868566f358ccd4dd36f84.png)](#)

## 结论

为每个环境手动编写部署脚本的日子正在迅速结束。在本文中，我们为 Oracle 数据库创建了一个完整的 CI/CD 管道。它仍然缺少一些关键特性，比如处理任何类型的初始化数据和静态数据，但是这是一个好的开始。我鼓励你采取这个基本的过程，并开始添加它。

下次再见，愉快的部署！

* * *

数据库部署自动化系列文章: