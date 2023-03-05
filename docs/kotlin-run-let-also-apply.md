# 为什么应该看看 Kotlin 的标准库——Octopus Deploy

> 原文：<https://octopus.com/blog/kotlin-run-let-also-apply>

随着我们在 Octopus 中增加对 Java 部署的支持，更多的集成代码正在用 Kotlin 编写。作为一名长期的 Java 开发人员，我借此机会了解了 Kotlin 语言设计者对他们的 Java 语言所做的一些改进。

大约在同一时间，我在 Pluralsight 上完成了戴夫·范彻(Dave Fancher)的 C# 函数式编程课程。我喜欢这门课，因为它提供了一些清晰而实用的建议，教你如何用 C#进行函数编程。在书中，Dave 提供了两个扩展方法，`Map`和`Tee`，这两个方法允许您转换对象并将它们传递给其他变异方法，并提供了如何以及为什么使用它们的示例。

我推荐这门课程，即使是对 Kotlin 开发人员来说，因为 Dave 提出的想法很好地映射了 Kotlin 作为其标准库的一部分提供的类似方法。

科特林`run`和`let`方法大致相当于 C# `Map`方法，而科特林`also`和`apply`方法大致相当于 C# `Tee`方法。

那么这些标准函数有什么区别呢？为了展示这些差异，我创建了一个简单的 Kotlin 项目，你可以在 GitHub 上找到它。

## 跑 vs 让

`run`和`let`是变换函数。它们接受被调用对象的值，并返回一个新值。

这些函数之间最明显的区别是它们暴露给块函数的变量。

`run`函数将调用它的对象的值公开为块内的`this`。

```
@Test
fun runExample () {
    val result = "Local String".run {
        System.out.println(this) // prints "Local String"
        "New String"
    }
    System.out.println(result) // prints "New String"
} 
```

`let`函数将调用它的对象的值公开为块内的`it`，而`this`从外部范围保留。

```
@Test
fun letExample () {
    val result = "Local String".let {
        System.out.println(this.name) // prints "Demo Class"
        System.out.println(it) // prints "Local String"
        "New String"
    }
    System.out.println(result) // prints "New String"
} 
```

您可以重命名默认的`it`参数。这样做可以避免默认的`it`参数与作用域中现有的变量发生冲突(如果嵌套两个`let`函数，就会发生这种情况)。

```
@Test
fun letExample2 () {
    val result = "Local String".let {me ->
        System.out.println(this.name) // prints "Demo Class"
        System.out.println(me) // prints "Local String"
        "New String"
    }
    System.out.println(result) // prints "New String"
} 
```

## 也 vs 适用

`also`和`apply`通常在调用它们所针对的对象的值需要用于某种变异操作时使用。来自`also`和`apply`模块的任何返回值都被忽略，并返回原始对象的值。

通过这种方式，我们可以利用原始值来执行一些变异逻辑(其返回值不被我们自己的代码使用)，同时保留原始值。

像`run`函数一样，`apply`将它所调用的对象的值公开为`this`。

```
@Test
fun applyExample() {
    val result = "Local String".apply {
        System.out.println(this) // prints "Local String"
        "New String" // return value is ignored
    }
    System.out.println(result) // prints "Local String"
} 
```

像`let`函数一样，`also`将调用它的对象暴露为块内的`it`，而`this`从外部作用域保留。

```
@Test
fun alsoExample() {
    val result = "Local String".also {
        System.out.println(this.name) // prints "Demo Class"
        System.out.println(it) // prints "Local String"
        "New String" // return value is ignored
    }
    System.out.println(result) // prints "Local String"
} 
```

并且`it`可以改名。

```
@Test
fun alsoExample2() {
    val result = "Local String".also { me ->
        System.out.println(this.name) // prints "Demo Class"
        System.out.println(me) // prints "Local String"
        "New String" // return value is ignored
    }
    System.out.println(result) // prints "Local String"
} 
```

## 转化与突变

我将`run`和`let`描述为转换函数，将`also`和`apply`描述为变异函数。

正如您在前面的例子中看到的，`run`和`let`也很乐意让您在它们的功能块中改变状态(就像我们通过写入控制台所做的那样)，因此转换和改变之间的区别是概念性的，而不是由语言强制的。

然而，这种概念上的区别是有用的，因为它允许您描述代码的意图，允许您简单地从代码的“形状”中了解代码做什么。

## 代码的形状是什么？

那么我所说的代码的“形状”是什么意思呢？让我们来看一个设计用于部署 AWS CloudFormation 模板的简单函数。

```
fun createCloudFormationStack(newStackName: String) {
  val credentialsProvider = ProfileCredentialsProvider()
  val client = AmazonCloudFormationClientBuilder
          .standard()
          .withCredentials(credentialsProvider)
          .withRegion(Regions.US_EAST_1)
          .build()

  val templatePath = javaClass.classLoader.getResource("WordPress.template.json").file
  val templateFile = File(templatePath)
  val template = FileUtils.readFileToString(templateFile, Charset.defaultCharset())

  val request = CreateStackRequest()
  request.stackName = newStackName
  request.templateBody = template

  val response = client.createStack(request)
  System.out.println("StackID: " + response.stackId)
} 
```

现在让我们取出所有的函数调用，看看代码是什么形状。

```
val credentialsProvider = ...
val client = ...

val templateFile = ...
val template = ...

val request = ...

val response = ... 
```

从这个代码的“形状”我们可以确定什么？

变量名给了我们一些关于我们正在创建的对象种类的指示，并且变量的顺序确实限制了对象之间的依赖种类。比如`client`可以依赖`credentialsProvider`，也可以不依赖`credentialsProvider`，但是`credentialsProvider`可以不依赖`client`，因为`credentialsProvider`是先声明的。

否则，虽然没有太多迹象表明这段代码在做什么。

让我们看一个使用标准库函数的版本。

```
fun createCloudFormationStack2(newStackName: String) {
    val client = ProfileCredentialsProvider().let { credentials ->
        AmazonCloudFormationClientBuilder
                .standard()
                .withCredentials(credentials)
                .withRegion(Regions.US_EAST_1)
                .build()
    }

    val template = javaClass.classLoader.getResource("WordPress.template.json").file.let { path ->
        File(path)
    }.let { templateFile ->
        FileUtils.readFileToString(templateFile, Charset.defaultCharset())
    }

    CreateStackRequest().also { request ->
        request.stackName = newStackName
        request.templateBody = template
    }.let { request ->
        client.createStack(request)
    }.also { response ->
        System.out.println("StackID: " + response.stackId)
    }
} 
```

去掉所有的函数调用(只留下标准函数)，我们得到这样的代码。

```
val client = ... .let { credentials ->
    ...
}

val template = ... .let { path ->
    ...
}.let { templateFile ->
    ...
}

... .also { request ->
    ...
}.let { request ->
    ...
}.also { response ->
    ...
} 
```

这个形状告诉我们什么？

*   我们可以看出，名为`credentials`的东西被转换(通过`let`函数)为分配给`client`的值。
*   我们可以看出，名为`path`的东西被转换为名为`templateFile`的东西，后者又被转换为分配给`template`的值。
*   我们可以看出，一个叫做`request`的东西发生了变异(通过`also`函数)，然后转化成了一个叫做`response`的东西。
*   我们可以看出`response`被用于一个变异函数调用中。

## 标准功能的好处是什么？

我们可以从代码中提取更多的上下文。

对象的转换被清楚地描述为:

*   `credentials` - > - `client`
*   `path` - > - `templateFile` - > - `template`
*   `request` - > - `response`

因为标准函数允许我们将一些变量的范围缩小到单个块，所以可以使用的对象组合明显更少。`template`变量的构造可以利用`client`(即使我们不这样做，代码的形状也不会阻止它)，并且`client`和`template`可以在创建`request`的最后一个块中使用(它们就是这样)。

将它与原始代码进行比较，在原始代码中，6 个变量中的每一个都可以利用它们的任意组合。只看第一个代码示例的形状，我们不知道所有变量是如何相关的。

减少变量也使得代码更容易推理。[这个神奇的数字](https://en.wikipedia.org/wiki/The_Magical_Number_Seven,_Plus_or_Minus_Two)描述了人类记忆能力的极限，介于 5 和 9 之间。虽然这个数字不是一个硬性规定，但根据我自己的经验，它听起来是真实的。

第一版代码中的 6 个变量意味着该功能正在突破普通人记忆的极限。第二个版本中的 3 个变量，加上一两个额外的变量，当我们移入和移出`let`和`also`函数时，将这段代码很好地放在普通人的记忆容量内。

由于对`also`的调用，我们也能够快速识别两个变异函数。这让我们知道测试这段代码需要做多少额外的工作，因为变异函数通常表明存在需要在测试环境中模拟和验证的外部状态。

在我们的例子中，第一个变异函数是在没有构建器接口的对象上设置参数，并且不接受构造器中的属性。这不需要跟踪外部状态。

然而，第二个变异函数写入控制台，根据应用程序的上下文，这对于测试可能重要，也可能不重要。

## 结论

虽然它们非常简单，但 Kotlin 中的标准函数提供了一种强大的方式来描述代码的意图。它们将减少变量计数，使代码更容易推理，并突出突变。