# 我们如何使用 SQL Server 作为文档存储- Octopus Deploy

> 原文：<https://octopus.com/blog/sql-as-document-store>

18 个月前，我在博客上写了我们如何决定在 Octopus 3.0 中从 RavenDB 转换到 SQL Server。那篇文章解释了我们遇到的问题以及我们决定转换的原因。我们随后发布了一篇关于[我们计划如何使用 SQL Server](https://octopus.com/blog/how-we-are-using-sql) 的帖子。Octopus 3.x 已经投入生产有一段时间了，所以在这篇文章中，我想提供一个更新，并详细介绍我们如何使用 SQL Server 作为文档存储。

## 目标

使用 Raven 时，我们发现文档数据库模型比关系模型更容易处理数据。尽管我们决定切换到 SQL Server，但我们希望保留许多我们在使用文档存储时已经习惯的方面。

我们心中有几个目标:

*   将整个聚合根加载/保存到单个表中
*   快速查询
*   普通的老酸，不需要担心最终的一致性
*   尽可能避免对连接的需求

## 典型表格结构

我们处理的每种类型的实体/聚合根(机器、环境、发布、任务)都有自己的表，遵循相似的模式:

*   一个 ID(详情如下)
*   我们查询的任何其他字段
*   JSON `nvarchar(max)`专栏

例如，这里有一张`Deployment`表。我们经常需要执行查询，如“哪些部署属于此版本”，或“此项目/环境的最新部署是什么”，因此表中有以下字段:

![Deployments table](img/789eaa2dc69d72a29f12af8eb6010cb2.png)

JSON 列包含文档的所有其他属性，包括任何嵌套对象。我们使用定制的 JSON.NET[契约解析器](http://www.newtonsoft.com/json/help/html/contractresolver.htm)来确保已经保存在表的其他列中的属性不会在 JSON 中重复。

## 关系

我们以不同的方式建立关系模型。对于我们*不*查询的关系，属性或数组只是在 JSON 中序列化。

当我们确实需要查询一个关系时，对于一对多的关系，如您所料，属性存储在一个列中:

![One-to-many](img/4902302804da72cf9c8d9c4d409d4273.png)

对于多对多，我们做了一些不太好的事情:我们存储每个值之间带有`|`字符的值:

![Many to many](img/2355046f9f89d22525d22bfd3d48383d.png)

这给了我们两个查询选项。对于我们期望只得到几十个、不超过一千个结果的表，我们可以简单地使用`LIKE '%|' + @value + '|%'`查询它们。我们知道它会导致全表扫描，但是这些查询很少，表很小，对我们来说已经足够快了。

对于增长到数百万条记录的表(审计事件，其中每个事件引用许多相关文档)，我们使用触发器。每次写入行时，我们都会分割值，并使用`MERGE`语句来更新连接表:

```
ALTER TRIGGER [dbo].[EventInsertUpdateTrigger] ON [dbo].[Event] AFTER INSERT, UPDATE
AS
BEGIN
    DELETE FROM [dbo].EventRelatedDocument where EventId in (SELECT Id from INSERTED);
    INSERT INTO [dbo].EventRelatedDocument (RelatedDocumentId, EventId)
        select Value, Id from INSERTED
        cross apply [fnSplitReferenceCollectionAsTable](RelatedDocumentIds)
END 
```

然后，我们可以对这个表执行连接查询，但是应用程序代码仍然可以将对象作为单个 JSON 文档加载/保存。

## ID 生成

我们使用字符串和类似于[高低算法](http://stackoverflow.com/questions/282099/whats-the-hi-lo-algorithm)的东西，而不是使用整数/标识字段或 GUIDs 作为 ID。格式总是`<PluralTableName>-<Number>`。

这是从 RavenDB 借鉴来的一个想法，它意味着如果我们有一个像审计事件这样的文档，它与许多其他文档相关，我们可以这样做:

```
{
   "Id": "Events-245",
   "RelatedDocumentIds": [ "Project-17", "Environments-32", "Releases-741" ]
   ...
} 
```

如果我们只是使用数字或 GUIDs，那么很难判断出`17`是项目的 ID，还是环境的 ID。将文档名作为 ID 的前缀有很多好处。

为了生成这些 id，我们使用一个名为`KeyAllocation`的表格。每个 Octopus 服务器使用一个可序列化的事务，通过增加下表中分配的范围来请求一个 ID 块(通常 20 个左右):

![KeyAllocation table](img/dc2450288a65ccab5e826022e78c5580.png)

随着时间的推移，服务器可以从批中发出这些 ID，而不必每次都去数据库。

## 来自的用法。网

使用中的数据库时。NET 代码，我们希望对数据库操作非常谨慎。大多数 ORM 都提供了一个[工作单元](http://martinfowler.com/eaaCatalog/unitOfWork.html) (NHibernate 的`ISession`，Entity Framework 的`DbContext`)，它跟踪对对象所做的更改，并自动更新已更改的记录。

由于工作单元的意外后果在过去一直是我们的一个性能问题，我们决定非常慎重地考虑:

*   交易边界
*   插入/更新/删除文档

我们只是抽象了一个 SQL 事务/连接，而不是一个工作单元。加载、编辑和保存文档的典型代码如下所示:

```
using (var transaction = store.BeginTransaction())
{
    var release = transaction.Load<Release>("Releases-1");
    release.ReleaseNotes = "Hello, world";
    transaction.Update(release);
    transaction.Commit();
} 
```

当查询时，我们不使用 LINQ 或任何花哨的标准/规范生成器，我们只是有一个非常简单的查询助手，它只是帮助构造 SQL 查询。您可以在 SQL 中做的任何事情都可以放在这里:

```
var project = transaction.Query<Project>()
    .Where("Name = @name and IsDisabled = 0")
    .Parameter("name", "My Project");
    .First();

var releases = transaction.Query<Release>()
    .Where("ProjectId = @proj")
    .Parameter("proj", project.Id)
    .ToList(); 
```

因为我们的表存储整个文档，所以对象/关系映射很容易——我们只需要指定给定文档映射到的表，以及哪些字段应该存储为列而不是 JSON 字段。下面是一个映射示例:

```
public class MachineMap : DocumentMap<Machine>
{
    public MachineMap()
    {
        Column(m => m.Name);
        Column(m => m.IsDisabled);
        Column(m => m.Roles);
        Column(m => m.EnvironmentIds);

        Unique("MachineNameUnique", "Name", "A machine with this name already exists. Please choose a different name. To add the same machine to multiple environments, edit the machine details and select additional environments.");
    }
} 
```

(虽然我们没有使用很多外键，但是我们依赖于一些约束，比如上面显示的唯一约束)

## 迁移

任何数据库，甚至是文档数据库，都必须在某个时候处理模式迁移。

对于一些变化，比如引入一个我们不打算查询的新属性，我们什么也不用做:JSON 序列化程序将自动开始编写新属性，并将处理旧文档中缺少的属性。类构造函数可以设置合理的默认值。

其他更改需要迁移脚本。对于这些，我们使用 [DbUp](https://dbup.github.io/) 。这要么是一个 SQL 脚本，要么是 C#代码(DbUp 可以让你任意执行。NET 代码)进行更高级的迁移。我们将很快在这个数据库的基础上完成四个主要版本和几十个次要版本，到目前为止还没有发现我们不能轻松处理迁移的情况。

## 统计数据

使用 SQL Server 作为文档存储对我们来说是可行的，但我确信它并不适合所有人。一些粗略的统计数据可能有助于正确看待这个问题:

*   自从 3.x 发布以来，已经有大约 15，000 次成功的 Octopus 安装——约占所有安装的 68%
*   其中一些安装非常大:将 300 多个项目部署到 3，500 多台机器上
*   有些在数据库中有成千上万的部署
*   最大的表是审计事件表，通常有数百万条记录
*   自从 3.x 发布以来，我们很少收到关于性能的投诉，性能问题也很容易得到解决

我们不仅在 Octopus 中使用这种方法——我们还在我们的网站和订单处理系统中使用这种方法。

## SQL Server 2016 JSON 支持

我们所遵循的方法将会变得更加普遍。 [SQL Server 2016 引入了对 JSON](https://blogs.msdn.microsoft.com/jocapc/2015/05/16/json-support-in-sql-server-2016/) 的支持，不是通过添加新类型(支持 XML 的方式)，而是通过添加函数来处理存储在`nvarchar`列中的 JSON。我们当前的模式完全可以利用这一点。

不幸的是，我们需要支持 2016 年以前的 SQL Server 版本，所以我们需要一段时间才能依赖这些功能。但是我们也许可以用它来优化某些情况——例如，如果 JSON 函数在您的 SQL Server 版本中可用，我们也许可以在服务器中做更少的处理。

## 为什么不用实体框架呢？

以下是我们使用 SQL 作为文档存储，而不是实体框架和关系存储的一些优势:

*   否[选择 N+1 个问题](http://www.codeproject.com/Articles/102647/Select-N-Problem-How-to-Decrease-Your-ORM-Perfor)；因为我们从单个表中获取整个对象树，所以不需要执行选择或连接来获取子记录
*   没有连接。在 Octopus 中有一个单一的代码路径，在这里我们连接两个表；在所有其他操作中，我们完全避免了连接。
*   轻松映射
*   您编写的查询就是您得到的查询。很容易推断出将要执行的 SQL 我们不依赖 LINQ 的翻译
*   多对多的关系更容易处理

## 摘要

改变存储技术和重写大部分应用程序来做到这一点是我们几年前下的一个大赌注，我对结果非常满意。性能更好，很少出现支持问题(不像以前的体系结构)，而且这是一种我们的客户更容易接受的技术。

虽然许多应用程序偶尔会使用 JSON 列来存储一些数据，但当我们开始这样做时，我还没有见过许多应用程序如此假装 SQL Server 是一个文档存储库，而忽略了大多数关系方面。不过，我确实认为这将变得更加普遍。最近在[上。网石！，杰瑞米·米勒谈到了 Marten](https://www.dotnetrocks.com/?show=1268) ，这听起来就像我们所做的，除了在 PostgreSQL 之上。随着 SQL 2016 获得自己的 JSON 函数，我预计它会变得更加流行。

我确信这种方法并不适合所有人。写入大量数据而查询较少的应用程序可能会从关系模型中获益更多。需要水平扩展数据库的应用程序应该坚持使用为该工作而构建的文档数据库。但是，如果您所在的环境中已经有 SQL Server，并且性能要求不是很高，那么 SQL 作为一个文档存储可能就适合您。