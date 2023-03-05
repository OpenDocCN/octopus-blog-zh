# 最终一致性的用户界面设计——Octopus 部署

> 原文：<https://octopus.com/blog/designing-for-eventual-consistency>

[Octopus Deploy 使用 RavenDB](http://octopusdeploy.com/blog/how-we-use-ravendb) 进行数据存储。项目、发布、部署、部署日志等等都保存在嵌入式 RavenDB 数据库中。Octopus 的 UI 是用 ASP.NET MVC 编写的，大部分数据呈现都是用 Razor 完成的。

这些技术选择实际上造成了一点问题: [RavenDB 索引是异步的](http://ravendb.net/docs/2.0/client-api/querying/stale-indexes)，但是我们真的希望在呈现页面时尽可能呈现最新的信息。RavenDB 应用程序应该为最终的一致性而设计，但是标准的 ASP.NET MVC 视图/控制器会让你认为你总是在呈现最新的信息。

![I love consistency](img/af4cecd65e16dc48236249db47574d85.png)

当我们第一次发布 Octopus 的 RavenDB 版本时，我们经常遇到这种错误报告:

> 我转到“添加机器”页面，输入机器信息，单击保存，但是当我转到“环境”页面时，我的机器没有列出。几秒钟后，我点击刷新，它就在那里。

为了解决这些问题，我将它分散在几乎所有的查询中:

```
.Customize(x => x.WaitForNonStaleResultsAsOfNow()) 
```

事实上，我做得太多了，以至于我创建了一个扩展方法来简化它:

```
.WaitForAges() 
```

这样做的好处是每个页面总是显示最新的信息。缺点是，一旦服务器有太多的文档，请求就会超时，因为索引已经过时了。这正在成为一个问题。

## 为最终一致性而设计

对于 Octopus Deploy V2 来说，最终一致性模型不是一个弱点，而是一个优势。

当我们呈现一个页面时，我们将避免对 RavenDB 进行任何查询。相反，我们将呈现页面所需的 HTML/CSS/JS。对于页面上的每个区域，我们将使用 [SignalR](http://signalr.net/) 异步获取数据。结果将包括索引是否过时，如果是，我们将让用户知道(“该数据可能已经过时——最后更新于 XYZ”)。

然后，使用 SignalR，我们将订阅 RavenDB 中的[更改通知，并将它们转发回应用程序，告诉它数据已经更改。](http://ayende.com/blog/157121/awesome-feature-of-the-day-ravendb-changes-api)

一旦页面的某个区域知道有新数据可用，它可能会:

*   立即显示更新
*   显示消息/刷新按钮

这种方法应该有望让 UI 感觉更快(因为我们可以快速呈现信息，而无需等待索引更新)以及感觉更实时(不再需要点击 F5)。