# HCL 和 HCL 工具简介- Octopus 部署

> 原文：<https://octopus.com/blog/introduction-to-hcl-and-hcl-tooling>

[![A laptop screen displaying analytics data that is peeled up on one corner showing the code behind it](img/a6098e741b1052b05b31f3c8e78f3f16.png)](#)

[HashiCorp 配置语言(HCL)](https://www.terraform.io/docs/configuration/index.html) 是一种独特的配置语言。它被设计用于 HashiCorp 工具，特别是 Terraform，但是 HCL 已经扩展为一种更通用的配置语言。它在视觉上类似于 JSON，并内置了额外的数据结构和功能。

HCL 由三个子语言组成:

*   结构的
*   表示
*   模板

当组合起来时，这些子语言形成了一个结构良好的 HCL 配置文件。这种结构有助于准确、轻松地描述 Terraform 工具所需的环境配置。

最近，HCL 已经放弃使用该语言的版本 1，转而使用版本 2。这篇文章假设我们谈论的是 HCL2 而不是 HCL1。

HCL2 是 HCL 和 HashiCorp 插值语言(HIL)的组合。HIL 增加了字符串插值和更强的在变量声明中使用函数的能力。

除了 Terraform 之外，HCL 还可以用于其他工具。随着时间的推移，不同的解析器变得可用，比如 Go、Java 和 Python。在这篇文章中，我将讨论如何开始使用 HCL，以及哪些工具可以利用它的独特特性。

## HCL 语言和特性

HCL 是一种 JSON 兼容语言，它增加了一些功能来帮助您最大限度地使用 Terraform 工具。这些特性使 HCL 成为一种强大的配置语言，并解决了 JSON 的一些缺点。

*   注释有单行和多行两种形式:
    *   单线:`#`或`//`。
    *   多行:`/* */`(无块注释嵌套)。
*   变量赋值使用`key = value`结构，其中空格无关紧要，值可以是一个原语，比如字符串、数字、布尔值、对象或列表。
*   字符串用引号括起来，可以包含任何 UTF-8 字符。
*   数字可以用多种不同的方式书写和解析:
    *   十进制数是默认值。
    *   十六进制:用`0x`作为数字的前缀。
    *   八进制:在数字前加一个`0`。
    *   科学数字:使用符号，如`1e10`。
*   使用`[]`创建数组和`{ key = value }`创建列表很容易。

这个概述只是触及了 HCL 的皮毛。通过研究一个示例配置文件并分析其工作原理，可以更容易地了解 HCL 的工作原理。

## 创建简单的 HCL 配置文件

为了了解 HCL 配置的实际情况，让我们创建一个简单的配置，演示一些可用的功能:

```
/*
Define the default configuration values here
*/
default_address = "127.0.0.1"
default_message = upper("Incident: ${incident}")
default_options = {
  priority: "High",
  color: "Red"
}

incident_rules {
    # Rule number 1
    rule "down_server" "infrastructure" {
        incident = 100
        options  = var.override_options ? var.override_options : var.default_options
        server   = default_address
        message  = default_message
    }
} 
```

你可能注意到我们对`default_message`变量使用了函数调用和字符串插值。`upper()`函数将使字符串大写，而`${}`构造用给定字符串中的值替换其中的变量。

与其他配置语言相比，另一个突出的特性是`rule "down_server" "infrastructure"`格式。这是一个`type label label`格式。在这个例子中，我们定义了应用程序中使用的规则类型和规则类别。

你也可以看到我们为`options`变量使用了一个三元条件。如果`override_options`变量存在，我们就使用那个值，否则我们就使用`default_options`。这种配置展示了 HCL 利用逻辑、字符串插值和语言内部操作的强大能力。

## 轻松编辑 HCL 配置

Visual Studio Code (VS Code)是目前最流行的编辑器之一，由微软免费提供。VS 代码有扩展，为基本编辑器增加了额外的功能。其中包括:

*   一个 [HCL 扩展](https://marketplace.visualstudio.com/items?itemName=wholroyd.HCL)来提供适当的语言着色。
*   [Terraform 扩展](https://marketplace.visualstudio.com/items?itemName=4ops.terraform)也增加了 HCL 支持，尽管是以 HashiCorp 工具之一命名的。这个扩展提供了语法突出显示和基本验证。

Atom 编辑器是 VS 代码的替代编辑器，也很受欢迎。它还提供了一个 [HCL 语法高亮](https://atom.io/packages/language-hcl)包。

到目前为止，我已经讨论了在 HashiCorp 工具的上下文中使用 HCL，但是还有其他工具使用 HCL 文件用于不同的应用程序。

一个例子是 [hclq 命令行处理器](https://hclq.sh/)。这个命令行处理器提供了以下功能:

*   配置的检查和验证。
*   用`grep`或`sed`解析文件的替代方法。
*   值插值的预处理。

请注意，该工具目前没有 HCL2 支持，但它是计划中的。

许多组织现在使用 Go 编程语言，因此在 Go 程序中使用 HCL 可能是一个有吸引力的选择。有一个 [Go 包](https://godoc.org/github.com/hashicorp/hcl)可以把 HCL 解码成可用的 Go 结构。

该模块使用 HCL 配置，并将输入解码为抽象语法树(AST ),从而可以在 Go 中轻松操作配置。将 HCL2 集成到服务器端 Go 程序中，可以让你使用简单易懂的复杂配置。

对于 Python 来说，有一个 [HCL2 解析器](https://pypi.org/project/python-hcl2/)不支持 HCL1，但是对于大多数项目来说这是不必要的。Python HCL2 使用解析工具包 Lark 构建，使得将 HCL2 配置集成到任何 Python 项目中变得容易。

## 结论

HashiCorp 配置语言最初是特定于 HashiCorp 的，但已经发展到在各种项目中变得更有吸引力。

最近的 HCL2 重写进一步合并了字符串插值和附加函数。这增加了已经很灵活的语言的可用性。易于理解的内置模板化的配置语言的强大功能，正在迅速使 HCL2 成为复杂配置的首选语言。

愉快的部署！

* * *

Adam Bertram 拥有 20 多年的 IT 经验，是一名经验丰富的在线商务专家。他是多家科技公司的顾问、微软 MVP、博客作者、培训师、出版作家和内容营销人员。在 adamtheautomator.com 的[网站](http://adamtheautomator.com/)上关注亚当的文章，在 LinkedIn 的[网站](https://www.linkedin.com/in/adbertram)上联系，或者在 Twitter 的 [@adbertram](https://twitter.com/adbertram) 上关注他。