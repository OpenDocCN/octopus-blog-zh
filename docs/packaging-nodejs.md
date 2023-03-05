# 打包 Node.js 应用程序- Octopus Deploy

> 原文：<https://octopus.com/blog/packaging-nodejs>

Node.js 已经成为近年来最热门的编程语言之一。JavaScript 已经连续几年在[栈溢出开发者调查](https://insights.stackoverflow.com/survey/2018)中被评为最受欢迎的编程语言，不难看出为什么。

JavaScript 作为 web 前端语言无处不在，这意味着无论开发人员来自 Java、C#还是 Ruby 背景，都可以使用和理解它。已经了解 JavaScript 的开发人员的数量意味着社区中挤满了各种可能需求的库和框架，因此有理由认为将它用于服务器端项目通常是正确的方法。

## 事情出错了

出于某种原因，当从编译语言转向脚本语言时，许多程序员似乎把他们的好习惯扔在了门外。在 Node.js 应用程序(或使用 Node.js 作为开发环境的纯 JavaScript 应用程序)中，开发人员将他们的代码提交给 GitHub，然后将源代码直接从 GitHub 拖到他们的生产服务器上，后面跟着一个`npm install`和`<grunt|gulp|webpack> build`，这种情况太常见了。通常，他们足够勤奋地编写测试，但决定在每次部署时重新构建应用程序更容易。这可能会导致几个问题。

### 依赖版本控制

通过在您的`packages.json`文件中使用 semver 范围，您可能会遇到这样的情况:您的配置要求某些包依赖项需要与版本`~4.17.2`匹配，因此您的开发机器关闭了`4.17.3`，针对`4.17.4`运行了测试，并且在部署产品时，库作者已经进行了另一次更新，并且您的产品软件正在运行`4.17.9`。我再说一遍，生产中！有很多这样的例子，库作者偶然地(或者没有意识到他们的改变的影响)用相应的版本碰撞来推动突破性的改变，由于广泛的版本匹配导致消耗代码崩溃。在`node_modules`中可能出现的复杂依赖链加剧了这个问题。例如，当一个显式依赖的模块 A 依赖于模块 B 的某个版本范围时，模块 B 本身依赖于一堆其他模块的某个版本范围。记住当你运行`npm install`时，不仅仅是*你的*模块被动态确定和解析。

npm 5 中引入的 [`package-lock`](https://docs.npmjs.com/files/package-lock.json) 特性在一定程度上缓解了这个问题。**确保将这个文件提交给源代码管理**。

### 部署成功在您的控制之外

每次部署应用程序时，您都在下载新的、潜在的未经测试的依赖项！这是假设一切顺利。尽管 npm 不再支持将新文件推送到现有版本，但我听说过不止一次生产发布失败的情况，因为由于 npm 暂时关闭或网络问题，包无法下载。这种安装失败通常伴随着覆盖旧版本，导致服务器在恢复对 npm 的访问之前实际上毫无价值(或者更糟)。在一个案例中，每当用户的 AWS Elastic Beanstalk 在现有服务器场负载过重的情况下试图启动新的 web 服务器时，用户就执行这个*全新安装*过程。由于 npm 服务器本身经历了停机，新的服务器无法上线，并且无法处理一些流量，最终导致现有的、以前很好的网站关闭。这是在之后的*，最初的部署已经在*成功*了很多天，并且相同的产品版本被部署到新的服务器上。*

## 打包 Node.js 应用程序

希望我们都同意 JavaScript 应用程序应该像其他编译语言一样被对待，并在构建时打包成一个可部署的工件。

假设我们正在开发 [RandomQuote-JS](https://github.com/OctopusSamples/RandomQuotes-JS) 虚拟应用程序，一个简单的构建步骤可能是:

```
> npm install
> gulp build    # insert build tool of choice
> zip -r ../RandomQuotes.1.0.zip .
# ../RandomQuotes.1.0.zip created (5.2 MB) 
```

然后，可以在我的生产服务器上提取生成的 zip 文件，使应用程序可以按原样运行。然而，在本例中，我们压缩了整个应用程序目录，其中大部分包含我用于调试和编译的各种库。一旦所有的东西都构建好并准备好在生产中运行，我就不需要所有的 typescript 编译器、webpack 库和测试框架了，但是我们如何将开发时依赖项与生产依赖项分开呢？

如果您运行 [`npm prune --production`](https://docs.npmjs.com/cli/prune) ，npm 将从`node_modules`中移除在`package.json`的`devDependencies`部分中指定的所有包。即使您不使用我们的工具，我也建议您在将整个项目文件夹压缩到 CD 系统的归档文件中之前运行这个命令，无论您选择的部署工具是 Octopus Deploy、VSTS、Chef 还是一些手动过程。

为了使这个过程更简单，Octopus Deploy 发布了一个 Node.js cli 工具 octopjs，用于创建 zip(或 tar.gz)归档文件，而不必执行修剪步骤。默认情况下，使用项目的`package.json`中的名称和版本来生成包名；但是，这些值都可以被覆盖。`Octojs`可以通过控制台直接使用，也可以作为构建过程的一部分导入并在代码中使用。

### 通过命令行打包

使用该工具最简单的方法是全局安装它:

```
> npm install -g @octopusdeploy/octojs 
```

现在，从任何项目内部都可以调用`octojs`来打包应用程序。假设我们的项目刚刚构建并经过测试，打包应用程序就像下面这样简单:

```
> octojs pack -O C:\Bin --gitignore --dependencies prod

Created package C:\Bin\RandomQuotes.1.0.0.zip (1.66 MB) 
```

生成的包的大小现在小了很多，包 ID 和版本是从项目本身生成的。有了现在生成的包，我们可以将它直接推送到 Octopus Deploy 内置提要进行部署:

```
> octojs push --package C:\Bin\ramdomquotes.1.0.0.zip --server http://octopusserver.acme.com --apiKey API-F2K29BA08AA123
Push response 201 - Created 
```

### 通过代码打包

或者，您可以使用同一个库来打包您的项目，并一步一步地完成代码:

```
var octo = require("@octopusdeploy/octojs");
octo.pack({dependencies: 'prod', bypassDisk: true, root: "."},
    (err, data) => {
            console.log("Uploading package: "+ data.name);
            octo.push(data.stream, {
                apiKey: 'API-F2K29BA08AA123 ',
                server: 'http://octopusserver.acme.com',
                name: data.name
            }, () => console.log("Uploaded"));
        }
    })
    .append('hello.txt', new Buffer('This Is An Additional File'))
    .finalize(true); 
```

## 让我们开始把 Node.js 当作一门严肃的语言来对待

Node.js 是一门严肃的语言，所以我们需要在我们的 CD 管道中严肃对待它。这意味着下载依赖项并在构建服务器上构建(或传输)一次，然后将结果和依赖项打包到一个自包含的部署包中。Octopus JS 库可以在这方面提供帮助，但最终您使用什么工具来打包和部署您的应用程序并不重要，重要的是它只需构建一次，就可以在您的环境中部署。

*更新:*查看[最近的一篇文章](https://octopus.com/blog/javascript-configuration)，在这篇文章中，我提供了一些如何在部署时获得特定于环境的配置的例子。

## 了解更多信息