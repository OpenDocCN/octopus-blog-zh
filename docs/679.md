# Selenium 系列:安装工具- Octopus Deploy

> 原文：<https://octopus.com/blog/selenium/28-installing-the-tools/installing-the-tools>

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。

到目前为止，我们一直在本地或在 Travis CI 平台上连续运行测试。然而，一个接一个地运行 UI 测试并不能很好地伸缩。随着更多测试的加入，以及更多组合和边缘案例的测试，我们经常会发现 UI 测试套件需要几个小时才能完成。

如果对 UI 测试有一个合理的批评，那就是它比其他形式的测试慢得多。如果没有更好的方式来运行 UI 测试，它们将成为 CI 管道中的瓶颈。那么我们能做些什么来使测试在更短的时间内完成呢？

> 一年前，我们的一个 UI 测试套件运行了几个小时。上个月，花了 16 分钟。今天，只需要 39 秒。

这是对亚马逊上的一篇博文的介绍，该博文讲述了一个开发团队如何使用 Chrome headless 和 AWS Lambda 将 UI 测试时间从数小时减少到数秒。这种测试时间的减少正是我们所期待的，在这篇文章中，我们将学习如何使用 AWS Lambda 大规模运行我们的 UI 测试。

Lambda 是 AWS 提供的一项服务，它允许短期应用程序以接近无限的规模运行。Lambda 负责提供操作系统和部署代码，您只需为应用程序实际运行的时间付费。Lambda 最好的特性是所有的缩放都由你负责。当你向 Lambda 中托管的应用程序发出更多请求时，平台将自动扩展以确保性能得到维持，如果请求减少，Lambda 将缩小规模。这种缩放是自动的，对我们的代码是透明的。

因为我们经常可以并发运行 UI 测试，所以通过 Lambda 托管我们的测试应用程序，我们可以显著减少执行测试套件所需的总时间，这要归功于这种自动伸缩。

让我们先来看看 AWS Lambda 控制台。从`Services`菜单中，点击`Lambda`链接。

[![C:\5d7441690ab4ca262ec5e50ff811e5c7](img/e7c3f799d24b6f2fca4883f56da5116d.png)](#)

因为我们还没有创建任何 Lambda 函数，Lambda 控制台提供了一个欢迎页面，在这里我们可以开始使用一些示例代码。

[![C:\2fd5e57c2c6430a4866aa46f08469702](img/88f961ea59c0fc98d348df2c3d5069cf.png)](#)

该页面中的代码是可编辑的，并且是一个真正的 Lambda 函数。这里的例子是用 JavaScript 编写的，但是即使您不熟悉 JavaScript，做一些简单的修改并观察它们是如何运行的也是很容易的。在下面的示例中，文本`Hello World!`已被替换为`This is a test of Lambda!`。点击`Run`按钮，然后在页面上显示这个 Lambda 的输出。

[![C:\e20d31044df9b3f11b74f4a809641eff](img/7c4c2dbe1e74fb73a69732571bdd3aa1.png)](#)

恭喜你！你刚刚跑完了你的第一个 Lambda。但这只是 JavaScript 代码，对吗？也许它正在浏览器中运行，这并不令人印象深刻。因此，让我们编写一些不能在浏览器中运行的代码来证明这些代码确实是在服务器上运行的。

与其用另一个字符串改变回调字符串，不如传递`process.platform`，它将返回代码运行的底层操作系统的名称。注意，我们得到的结果是`linux`。所以这段代码确实是在 Linux 服务器上运行，而不是在浏览器中。

[![C:\4a18d5261da767131748d47c718194bb](img/457167147e317aed5e650f6581454f00.png)](#)

那么运行代码的 Linux 服务器在哪里呢？这就是 Lambda 的美妙之处，我们不需要考虑服务器。AWS 负责确保服务器可用，无需管理。我们需要做的就是将代码部署为 Lambda 函数并调用它。AWS 会处理剩下的事情。

在我们开始之前，我们需要安装一些额外的工具来简化 Lambda 函数的部署。

我们将使用[无服务器](https://serverless.com/)平台将我们的 Lambda 函数部署到 AWS。Serverless 为我们提供了一些简单的命令，负责上传我们的代码，部署到 Lambda，以及设置 Lambda 所需的各种选项。

Serverless 是一个 Node.js 应用程序，这意味着我们需要在本地安装 Node.js 才能使用它。你可以从 https://nodejs.org/en/download/[下载并安装 Node.js。](https://nodejs.org/en/download/)

您可以通过从终端或 PowerShell 运行命令`node --version`来确认 Node.js 已安装:

```
$ node --version
v8.11.2 
```

然后无服务器应用程序安装有`npm`，它是节点包管理器的缩写。此命令会将无服务器应用程序下载并安装到您的本地计算机上。

在 Linux 或 MacOS 机器上，您可以使用`sudo`命令安装无服务器:

```
$ sudo npm install serverless -g 
```

在 Windows PC 上，您可以使用以下命令安装无服务器。

注意，您可能需要使用管理员权限从 PowerShell 或命令行窗口运行此命令:

```
$ npm install serverless -g 
```

然后，您可以通过运行`serverless --version`来确认无服务器命令是否可用:

```
$ serverless --version
1.27.2 
```

最后一项任务是用无服务器将用来访问 AWS 的密钥来配置您的系统。为此，我们需要打开 IAM 控制台。在 AWS 控制台中点击`Services`菜单，并点击`IAM`链接。

[![C:\4f8d073f4ff8baf251e365cd65cf2313](img/ef65364824511d6e66f54970495f5da8.png)](#)

点击`Users`链接。

[![C:\775c908af7799241eebaea2aeb89d097](img/7f021e9b3431bc439e1383beb641c990.png)](#)

点击`Add user`按钮。

[![C:\f240ce9cca4677631a2f672eb456c95b](img/b687e6c9a867074713e88fe7ffb70d69.png)](#)

在`User name`字段中为新用户命名，并选择`Programmatic access`选项。这个选项意味着用户将有能力生成一个访问密钥和秘密密钥，无服务器应用程序将使用这个密钥对来部署我们的代码。

点击`Next: permissions`按钮继续。

[![C:\1cfdcc00dd60c7c8695426caaa2b3896](img/b76e5564bcdf70f0ebafb84ce116b47a.png)](#)

我们将授予新用户管理员权限。为此，点击`Attach existing policies directly`按钮并选择`AdministratorAccess`策略。

点击`Next: review`按钮继续。

[![C:\f54d434c8f8ecec5370dc08c10d26824](img/a6da98e1f5539ca381ac9f4faf86cbf7.png)](#)

点击`Create user`按钮完成。

[![C:\2956bb9055ff7e3727479a9558081ff9](img/a8cd54d870b0767cd40de8c2b7e0a0cc.png)](#)

现在，您将看到新用户的访问密钥和秘密访问密钥。

这是你看到秘密访问密钥的唯一机会。一旦此屏幕关闭，就无法再查看此密钥，因此请务必将此密钥保存在安全的地方。

要查看秘密访问密钥，请单击`Show`链接。

[![C:\f1bbe175f49ba6bfcc320d2d73133828](img/650b4512852a6543570bc12aa6b7164e.png)](#)

请将这些密钥保存在安全的地方，以便以后使用。或者，您可以点击`Download .csv`按钮下载包含这些密钥的文件。

完成后，点击`Close`按钮。

[![C:\a38f73bc3cb7c2c394e207e1a5e92a90](img/4016cff2287fdc2e91cd124ebe92067e.png)](#)

为了允许无服务器应用程序使用这些凭证，我们需要在我们的用户主目录中创建一个名为`.aws/credentials`的文件。

*   在 Windows 中，该文件保存在`C:\Users\username\.aws\credentials`下。
*   在 Linux 中，这个文件保存在`/home/username/.aws/credentials`下。
*   在 MacOS 中，这个文件保存在`/Users/username/.aws/credentials`下。

该文件的内容应该如下所示，其中的键值替换为 AWS 用户的键值:

```
[default]
aws_access_key_id = AKIAJM25BFQYTRVXQSNQ
aws_secret_access_key = 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 
```

我们现在已经安装了无服务器应用程序并配置了 AWS 凭证，这意味着我们可以使用无服务器应用程序将代码部署到 AWS。然而，不像运行一个独立的 Java 应用程序作为 Lambda 函数，不需要太多额外的配置就可以工作，我们需要完成一些额外的步骤来提供一个可以运行 WebDriver 测试的环境，我们将在下一篇文章中解决这个问题。

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。