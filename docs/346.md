# 在配置中引入 slugs 作为代码- Octopus Deploy

> 原文：<https://octopus.com/blog/introducing-slugs-config-as-code>

当我们开始开发配置为代码特性时，我们决定用 Octopus 配置语言(OCL)中的名称替换 id。然而，这种方法有几个缺点，导致我们在 OCL 用鼻涕虫代替名字。

这篇文章探讨了在 OCL 使用名字时的权衡，什么是 slugs，以及为什么我们选择在 Config as Code 中实现它们。

## OCL 的身份证

传统上，Octopus 使用惟一的 id 来引用来自部署流程、操作手册和变量等地方的共享资源。

就其本身而言，id 不是非常具有描述性的，它们只是由资源类型和一个唯一的数字组成。例如，环境的 ID 可能类似于`Environments-42`。

Octopus 中的 id 缺乏上下文。光有一个 ID 还不足以知道这个 ID 指的是什么。对于 API 之类的东西来说，这不是问题，因为 API 响应通常被设计成由其他程序读取。

然而，OCL 是给人写和读的。在 OCL 使用 id 会让人一眼难以理解，如下例所示:

```
step "Run a Script" {
  action {
    environments = ["Environments-42"]
    worker_pool = "WorkerPools-4"
    ...
  }
} 
```

断章取义，不清楚这是干什么的。`Environments-42`是什么环境？`WorkerPools-4`是哪个工人池？这些 id 在很多地方都不公开，所以试图弄清楚它们指的是什么可能是一个挑战。

如果你只是想读 OCL，这不是一个很好的经历。

## OCL 的名字

在 OCL，我们没有使用 IDs，而是决定使用资源的名称。

Octopus 中的名字非常独特，我们可以放心地用它们来识别资源。名称也很容易一眼识别，使 OCL 更具可读性和用户友好性。

```
step "Run a Script" {
  action {
    environments = ["Staging"]
    worker_pool = "Hosted Ubuntu"
    ...
  }
} 
```

看看上面的 OCL，与在 OCL 使用 IDs 相比，更容易理解发生了什么。

在我们决定在 OCL 使用名字之后，我们也决定在 API 和 UI 中为版本控制的项目使用名字*来代替*id。

我们这样做是为了改善从 UI 中查看受版本控制的部署流程时的用户体验。如果 OCL 中有任何损坏的引用(比如名称中的打字错误)，我们会在发现损坏的引用的地方显示一个警告，并允许从 UI 或 API 修复它

虽然这种方法最初奏效，但它存在一些长期的缺点。

## 使用名称的权衡

在整个 Octopus 代码库中，我们通常假设所有对共享资源的引用都是使用 IDs 进行的。当我们在 OCL 引入名字时，我们打破了这个假设。

ID 属性可以包含一个有效的 ID *或*一个可能存在也可能不存在的资源的名称。

这导致了内部和外部的一些突破性的变化，因为*“names as id”*方法也影响了我们的 API，这意味着 API 消费者(包括我们自己)也必须知道响应中的名称和 id。

因为从技术上讲，id 作为名称也是有效的，所以可能不清楚给定的值是 ID 还是名称，让用户自己去猜测。

看看下面的 JSON，有几个问题:

*   属性名称以`Id`为后缀，但是，名称仍然存在。这是令人困惑和误导的。
*   不清楚`EnvironmentId`属性的值指的是什么。
    *   `Environment-1`是名字还是 ID？
    *   如果我有一个 ID 为`Environment-1`的环境和另一个名为`Environment-1`的环境，这是指哪一个呢？

```
{
  "EnvironmentId": "Environment-1",
  "WorkerPoolId": "Hosted Ubuntu"
} 
```

使用名称作为 id 还会导致喋喋不休的 API 客户端，因为消费者需要向 API 发出额外的请求才能获取名称引用的资源。

需要使用通用列表样式的端点(例如，`/environments?partialName=foobar`)来查找所有匹配的资源，然后需要消费者手动过滤结果。这些请求可能并不便宜，尤其是在进行分页并且需要对单个资源发出多个请求的情况下。

在 OCL，由名称引起的另一个小问题是无法在不破坏引用的情况下重命名资源。这是意料之中的，但仍有改进的余地。

## 介绍鼻涕虫

考虑到使用名字作为 id 的缺点，我们考虑使用 slugs。

Slugs 是人类可读的、URL 友好的、唯一的标识符。它们通常用在名称不太合适的地方，比如 URL(包括这篇文章的 URL)。

Slugs 通常是根据名称自动生成的。例如，一个名为`Test Environment (AU-EAST)`的环境会生成`test-environment-au-east`，删除任何不友好的字符，同时保持可读性。

## 章鱼的鼻涕虫

为了保持较小的范围，我们首先将 [slugs](https://octopus.com/docs/projects/version-control/config-as-code-reference#slugs-in-ocl) 添加到可以从部署流程中引用的任何内容中。迄今为止，这包括:

*   帐目
*   频道
*   部署行动
*   部署步骤
*   部署目标
*   环境
*   饲料
*   生活过程
*   组
*   工人池

随着需求的增加，我们将为更多的资源类型添加 slugs。

项目已经有了 slugs 的概念，它大多符合我们自己的目标。我们设法重新调整了现有项目 slug 逻辑的大部分用途，并通过一些调整将其应用于上述资源类型。

*   任何新创建的或现有的资源都会根据它们的名称自动生成它们的 slugs。
*   Slugs 可以独立于名称进行修改，允许在不破坏任何引用的情况下重命名资源。*这改变了项目段塞的现有行为。*
    *   通常不建议修改 slugs，因为这可能会导致需要手动更新的外部引用被破坏(例如，在 OCL 定义的部署流程)。

## OCL 的蛞蝓

在我们将 slugs 添加到所有必要的资源之后，我们开始在我们的 OCL 中使用 slugs。

这被证明是相对简单的，因为我们可以重用现有的 ID 来命名转换代码，并添加一些逻辑来在 ID 和 slugs 之间进行转换。

我们还更新了部署步骤和动作的 OCL 语法，这样 slugs 可以与名称分开指定。

```
step "run-a-script" {
  name = "Run a Script"

  action {
    environments = ["staging-au-east"]
    worker_pool = "hosted-ubuntu"
    ...
  }
} 
```

现在更容易理解它指的是哪个环境和工人池。我们还受益于这些 slugs 是独一无二的，可以在项目和空间之间重复使用。

## 隔离抽象泄漏

在撰写本文时，使用 slugs 引用共享资源只有在 OCL 才有意义。我们的前端和后端都没有准备好使用 slugs 来引用这些资源，更新它们来这样做将是一项重要的任务。

因为 slugs 可以用作上下文唯一的 ID，所以当从 Octopus 中读取和写入 OCL 时，我们可以在传统 ID 和 slugs 之间进行映射。这允许我们在 OCL 使用 slugs，同时在 Octopus 服务器、API 和前端使用 IDs。

如果 Octopus 遇到一个无法映射到 ID 的 slug，就会返回一个验证错误，而不是允许中断的引用。我们可以保持我们正在处理的数据的有效性。

虽然这牺牲了从 UI 和 API 查看和编辑中断的引用的能力，但我们相信这是正确的方向，因为它使我们的 API 恢复了一致的形状，并提高了 Octopus server 在与 OCL 一起工作时的稳定性和可预测性。

## 蛞蝓的未来

随着 [slugs](https://octopus.com/docs/projects/version-control/config-as-code-reference#slugs-in-ocl) 的大部分基础工作已经完成，我们可以开始向更多的资源类型添加 slugs，并开始在 Octopus 代码库中更多地使用它们。

Config as Code 之外的许多其他团队也对在其领域中使用 slugs 表现出了兴趣，所以请继续关注我们接下来如何使用 slugs。

愉快的部署！