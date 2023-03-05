# Octostache 更新- JSON，索引，格式化和日期- Octopus 部署

> 原文：<https://octopus.com/blog/octostache-json-formatting>

作为支持 Docker 部署的一部分，我们意识到我们需要一种方法来访问作为集装箱检查输出返回的 JSON 格式的信息。与其递归遍历 JSON 对象并在运行 Docker 步骤后将所有属性转换为 Octopus 变量，*以防后续步骤需要它，我们决定在需要变量*时更新[Octopus che](https://github.com/OctopusDeploy/Octostache)(我们的[变量替换库](http://docs.octopusdeploy.com/display/OD/Variable+Substitution+Syntax))来支持解析 JSON *。虽然我们已经接触了变量解析库，但是我们认为在 3.5 中增加一些其他有用的特性是一个很好的借口！*

### JSON 解析

例如，如果将项目或输出变量设置为 JSON 值:

```
Variables:
Custom.Json = "{Name: 't-shirt', Sizes: [{size: 'small', price: 15.00}, {size: 'large', price: 20.00}]}"  

Expressions:
#{Custom.Json.Name}
#{Custom.Json.Sizes[0].price}
#{Custom.Json.Sizes[1]}

Resolves to:
"t-shirt"
"15.00"
"{size: "large", price: 20.00}" 
```

请注意，如果您已经显式地提供了一个与将解析为 JSON 路径的值相匹配的变量，例如，如果您显式地设置了一个项目变量

```
Custom.Json.Name = "pants" 
```

那么将返回显式变量`pants`。这个新的解析支持如上所示的点符号或索引符号，因此`#{Custom.Json.[Sizes][0][price]}`也可以解析为`'15.00'`。这最后一点把我带到下一个更新...

### 索引替换

不久前，我们增加了在索引中执行变量替换的支持。

```
Variables:
Server[Asia] = "Beijing"
Server[Europe] = "London"
Continent = "Asia"

Expression:
#{Server[#{Continent}]}

Resolves To:
"Beijing" 
```

作为变量替换本身的结果，您现在可以动态解析适当的`Server`变量。顺便说一下，它对 Docker 步骤也很方便，您可能希望从容器 JSON 输出中访问一个变量，该变量由另一个步骤的一些属性输出索引，而这些属性输出只能在部署时知道。

### 条件式

直到最近，变量替换中的条件语句需要一个*真值*比较。也就是说，只有当变量的值类似于`"True"`时，它们才会将条件解析为`true`。现在不再是这种情况，比较现在可以检查直接字符串相等或使用内部变量替换。

```
Variables:
City = "London"
Capital = "London"

Expressions:
#{if City == "London"}'Ello Guvna#{/if}
#{if City != "Paris"}'Ello Guvna#{/if}
#{if City == Capital}'Ello Guvna#{/if}

Resolve To:
'Ello Guvna
'Ello Guvna
'Ello Guvna 
```

### 格式化

我们的一些用户面临的一个问题是，当我们将项目变量中的日期保存到数据库中时，序列化程序会将它们转换成不同的格式。我们引入了一个新的内置八进制函数，允许你将日期变量格式化为. NET 支持的任何自定义格式

```
#{<VariableName> | Format <DataType> <Format>} 
```

其中`<DataType>`可以是“DateTime”、“DateTimeOffset”、“Double”、“Decimal”或“Int”之一(不区分大小写)。Octostache 然后会尝试将在`<VariableName>`解析的变量解析为指定的类型，然后用 C# `.ToString()`方法转换回一个字符串，传入提供的`<Format>`参数。

```
Variables:
Cash = "12.1"
MyDate = "2030/05/22 09:05:00"
CustomFormat = "MMM dd, yyyy"

Expressions:
#{Cash | Format double C}
#{MyDate | Format DateTime \"HH dd-MM-yyyy\"}
#{MyDate | Format DateTime #{CustomFormat}}

Resolve To:
$12.1
09 22-May-2030
May 22, 2030 
```

请注意，格式化将使用进行转换的计算机的当前区域性。此外，由于这些参数由空格分隔，如果您的格式包含空格，就像上面的日期格式一样，您将需要包含引号。如果缺少`<DataType>`参数，格式化程序将首先尝试将变量解析为小数，然后解析为 DateTimeOffset。

除了新的`NowDate`和`NowDateUtc`函数(没有变量输入)，您还可以获得当前时间戳，并将其与格式化函数链接在一起。

```
Expression:
#{ | NowDate | Format yyyy}

Resolves to
"2016" 
```

### 结论

我们希望您会发现这些新的变量替换功能非常有用。Octostache，我们的变量替换库是开源的，并且已经在 GitHub 上提供了[，所以请随意查看并贡献您自己的增强建议，或者从我们的](https://github.com/OctopusDeploy/Octostache)[在线文档](http://docs.octopusdeploy.com/display/OD/Variable+Substitution+Syntax)中了解更多关于在您的部署中使用这些新表达式的信息。Octopus 3.5 的这些新增功能允许变量和作用域之间更丰富和复杂的关系。让我们知道它们如何(或不如何)对您的部署有用！