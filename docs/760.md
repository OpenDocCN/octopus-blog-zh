# 浏览器 UI 测试的注意事项- Octopus Deploy

> 原文：<https://octopus.com/blog/ui-testing-considerations>

[![Considerations for browser UI testing](img/fee2454a1cdf5ed4dfa1468c242f7a6a.png)](#)

通过网络浏览器模拟用户交互的端到端测试已经变得越来越普遍。WebDriver 现在是一个[开放的 W3 标准](https://www.w3.org/TR/webdriver/)，所有主流的 web 浏览器都提供了 WebDriver 接口。最近的项目如 [Cypress](https://www.cypress.io/) ，已经实现了他们自己的与浏览器交互的方法，专门用于编写测试。有了这样一系列得到良好支持的成熟测试平台，编写基于浏览器的测试变得前所未有的简单。

但是，在自动化 CI/CD 环境中运行基于浏览器的测试时，有一些注意事项需要考虑。在这篇博文中，我们将探讨如何运行这些测试以及测试环境的局限性。

## Windows 交互式服务

在我们开始使用 Windows 进行交互式测试之前，有必要了解一下交互式服务的历史。

如果您曾经配置过 Windows 服务，您可能已经注意到了`Allow service to interact with the desktop`选项:

[![](img/f2cee231e893d3c1d687439bdecd35d2.png)](#)

回到 Windows XP，该选项允许 Windows 服务在第一个登录用户的桌面上绘图，该用户被分配了会话 0。从安全角度来看，这并不理想，导致了像[粉碎攻击](https://en.wikipedia.org/wiki/Shatter_attack)这样的攻击。

为了解决此漏洞，Windows Vista 引入了一项更改，即在会话 0 中运行服务，在会话 1 及更高版本中运行用户。这意味着用户不再暴露于服务所绘制的任何界面，这为期望用户与对话框提示交互的服务带来了向后兼容性问题。

创建 Interactive Services 检测服务是为了方便与这些提示进行交互，显示如下消息:

【T2 ![](img/89af73d9fe8167e696ac41282d10d6bd.png)

也可以用命令`rundll32 winsta.dll,WinStationSwitchToServicesSession`切换到会话 0。

然而，最新版本的 Windows 10 已经[移除了互动服务检测服务](https://support.microsoft.com/en-au/help/4014193/features-that-are-removed-or-deprecated-in-windows-10-creators-update)并禁止任何对会话 0 的访问。这意味着用户现在完全脱离了交互式服务，没有第三方解决方案。

此外，允许服务与桌面交互的选项在默认情况下通过`NoInteractiveServices`注册表键[被](https://docs.microsoft.com/en-us/windows/win32/services/interactive-services)禁用。

底线是，虽然微软从未正式表示运行交互式服务不受支持或不被重视，但所有影响交互式服务的 Windows 更改都是为了限制或禁用它们的使用。即使您今天设法实现了一个交互式服务，假设它在将来会继续工作也是不明智的。

## 拯救无头浏览器

PhantomJS 推广了无头浏览器的概念。这使得自动化测试可以在非交互式环境中运行。

此后，Chrome/Chromium 和 Firefox 增加了对无头模式的支持。随着主流浏览器中的无头支持，PhantomJS 变得多余，并且[不再被维护](https://groups.google.com/forum/#!topic/phantomjs/9aI5d-LDuNE)。

尽管如此，仍有许多浏览器，如 Internet Explorer 和 Edge，不支持(也可能永远不会支持)无头模式。headless 模式也没有提供测试桌面应用程序用户界面或捕获屏幕记录的解决方案。那么，当你需要一个真正的桌面来进行测试时，有什么选择呢？

## Windows 交互式代理

对于 Windows，我们可以依靠 Azure DevOps 交互代理来指导微软如何支持在真实桌面上运行测试。微软的解决方案是给 Windows 机器配置[自动登录](https://support.microsoft.com/en-au/help/324737/how-to-turn-on-automatic-logon-in-windows)，然后在启动时运行代理启动 UI 测试。这样，被测试的应用程序出现在普通用户的桌面上。

但是这个解决方案[并非没有风险](https://github.com/MicrosoftDocs/vsts-docs/blob/master/docs/pipelines/agents/agents.md#interactive-or-service):

> 当您启用自动登录或禁用屏幕保护程序时，会有安全风险，因为您允许其他用户走近计算机并使用自动登录的帐户。如果将代理配置为以这种方式运行，则必须确保计算机受到物理保护；例如，位于安全设施中。

Octopus 的一个类似解决方案是在启动时用下面的命令运行触手(用本地触手的名称替换`instance`名称):

```
"C:\Program Files\Octopus Deploy\Tentacle\Tentacle.exe" run --instance="Tentacle" 
```

## Linux 无头桌面

使用 [xvfb](https://www.x.org/releases/X11R7.6/doc/man/man1/Xvfb.1.xhtml) ，Linux 用户可以更加灵活地在无头环境中运行应用程序。Xvfb 使用虚拟内存模拟一个哑帧缓冲区，这意味着所有应用程序都被渲染到内存中，而不需要任何显示硬件。

我们已经使用 xvfb 通过配置一个完整的 Linux 桌面环境来创建 [Octopus Guides](https://octopus.com/blog/devops-documentation) ，浏览器在这个环境中启动。这个 [webdriver](https://github.com/OctopusDeploy/WebDriverTraining/blob/master/docker/webdriver) 脚本展示了 xvfb 是如何启动的，并且定义了`DISPLAY`环境变量来实现这一点。

## 场外测试

有许多像 Browserstack 或 Sause Labs 这样的服务提供自动化测试服务。通过控制远程浏览器，您不再负责配置底层操作系统，大多数浏览器将提供视频录制等功能。

## 结论

在远程系统上运行自动化测试时，无头浏览器应该是您的首选。Linux 用户可以利用 xvfb 在无头桌面上运行应用程序。但是，如果你需要一个真正的桌面来测试 Windows 中的应用程序，启用自动登录并在启动时运行代理或触角是最好的解决方案。

使用交互式服务你可能会有一些运气，但这种选择是脆弱的，因为微软在每次更新 Windows 时都会限制和禁用交互式服务。