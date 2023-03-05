# Octopus Deploy 2.0 API 更改- RFC - Octopus Deploy

> 原文：<https://octopus.com/blog/octopus-2.0-api-rfc>

Octopus Deploy 拥有 RESTful HTTP API 已经有一段时间了，可从以下网址获得:

```
http://<your-octopus-server>/api 
```

该 API 主要是为了增强 Octo.exe 的功能而构建的，这是一个用于创建和部署版本的命令行工具。除了机器、版本和部署之外，大多数现有 API 都是只读的。当我们在 Octopus UI 中引入新的特性时，它们并不总是能融入到 API 中；公平地说，API 实际上是一个二等公民。

对于 Octopus Deploy 2.0，我们将对我们的 UI 做一些大的改变，我在 [UI 设计以实现最终一致性](http://octopusdeploy.com/blog/designing-for-eventual-consistency)中写了博客。我想确保 Octopus 即使在非常大的装置上也能感觉快速和流畅。此外，我们经常收到功能请求，要求能够在 API 中做一些我们目前只能通过 UI 做的事情。

因此，对于 2.0 版本，我们将使我们的 API 成为一等公民。UI 将利用 API 这意味着 UI 中的大多数交互将通过 API 发出异步请求，而不是直接发送到 MVC 控制器。

由于 API 得到了如此多的关注，我们也将在重新构建它时对它进行适当的文档化。为此，我在 GitHub 上创建了一个项目:

**[章鱼部署-API](https://github.com/OctopusDeploy/OctopusDeploy-Api) 在 GitHub 上**

这个项目目前还很少，但是你会发现有一部分是关于认证和我们使用链接导航 API 的方式。

如果您正在基于 Octopus API 进行构建，或者打算这样做，我建议您遵循这个存储库，并通过文档中的问题/拉请求提供反馈。