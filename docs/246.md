# 使用动态数量的参数从 PowerShell 调用可执行文件——Octopus Deploy

> 原文：<https://octopus.com/blog/dynamic-argument-list-when-calling-executable-from-powershell>

从 PowerShell 调用可执行文件很容易——大多数时候，您只需在前面加上一个`&`。为了说明，让我们看看这个 C#可执行文件:

```
static void Main(string[] args)
{
    for (int i = 0; i < args.Length; i++)
    {
        Console.WriteLine("[" + i + "] = '" + args[i] + "'");
    }
} 
```

如果我们这样称呼它:

```
& .\Argsy.exe arg1 "argument 2" 
```

我们得到:

```
[0] = 'arg1'
[1] = 'argument 2' 
```

PowerShell 变量也可以传递给参数:

```
$myvariable = "argument 2"
& .\Argsy.exe arg1 $myvariable

# Output:
[0] = 'arg1'
[1] = 'argument 2' 
```

注意,`$myvariable`的值包含一个空格，但是 PowerShell 很聪明，将整个值作为一个参数传递。

当您想要有条件地或动态地添加参数时，这就变得棘手了。例如，您可能会尝试这样做:

```
$args = ""
$environments = @("My Environment", "Production")
foreach ($environment in $environments) 
{
    $args += "--environment "
    $args += $environment + " "
}

& .\Argsy.exe $args 
```

然而，您会对输出感到失望:

```
[0] = '--environment My Environment --environment Production ' 
```

## 正确的方式

相反，这样做的方法是创建一个数组。您仍然可以在 PowerShell 中使用`+=`语法来构建阵列:

```
$args = @() # Empty array
$environments = @("My Environment", "Production")
foreach ($environment in $environments) 
{
    $args += "--environment"
    $args += $environment
}
& .\Argsy.exe $args 
```

它输出了我们期望的结果:

```
[0] = '--environment'
[1] = 'My Environment'
[2] = '--environment'
[3] = 'Production' 
```

您也可以将常规字符串与数组混合使用:

```
& .\Argsy.exe arg1 "argument 2" $args

# Output:
[0] = 'arg1'
[1] = 'argument 2'
[2] = '--environment'
[3] = 'MyEnvironment'
[4] = '--environment'
[5] = 'Production' 
```

## 边缘情况

对于我上面所说的传递一个带有所有参数的字符串，有一种非常奇怪的情况。以这个例子为例，它与上面的例子相似:

```
$args = "--project Foo --environment My Environment --environment Production"
& .\Argsy.exe $args

# Output: 
[0] = '--project Foo --environment My Environment --environment Production' 
```

要使它按预期工作，只需在第一个参数的**两边加上引号，行为就会完全改变！(反勾号是 PowerShell 的转义字符)**

```
$args = "`"--project`" Foo --environment My Environment --environment Production"
& .\Argsy.exe $args

# Output: 
[0] = '--project'
[1] = 'Foo'
[2] = '--environment'
[3] = 'My'
[4] = 'Environment'
[5] = '--environment'
[6] = 'Production' 
```

如果不引用第一个参数，行为不会改变:

```
$args = "--project `"Foo`" --environment My Environment --environment Production"
& .\Argsy.exe $args

# Output: 
[0] = '--project Foo --environment My Environment --environment Production' 
```

啊，PowerShell。总是充满惊喜！