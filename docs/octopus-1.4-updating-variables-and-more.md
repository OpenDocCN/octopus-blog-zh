# Octopus 1.4 更新了变量、克隆项目和其他修正- Octopus 部署

> 原文：<https://octopus.com/blog/octopus-1.4-updating-variables-and-more>

[Octopus Deploy 1.4](http://octopusdeploy.com/downloads/1.4.0.1589) 刚刚发布。它包括许多错误修复，以及对我们调用 PowerShell 脚本的方式的[改进。](http://octopusdeploy.com/blog/improving-powershell)

在这个版本中，我们还实现了一些我们已经有一段时间的最受欢迎的功能请求。

## 更新发布变量

查看版本时，**您现在可以更新版本**中的变量:

![Updating Octopus release variables](img/d4f7f976b6392b578d57a61ad504f199.png)

点击此按钮将当前项目变量重新快照到版本中。如果您的某个步骤有新的变量，而这些变量在发行版中并不存在，那么这些变量将被跳过。还将捕获一条审计消息，以跟踪变量被更新的事实，以及由谁更新的。

只有在安装 Octopus Deploy 1.4 之后创建的版本才支持该特性，因为它需要对模式做一个小的更改，以允许导入变量。

## 克隆项目

需要复制一个项目？只需转到项目设置选项卡，然后单击此按钮:

![Cloning a project](img/aeb7e874e496ae016d7e4f73767dfbf5.png)

## 禁用机器

有时，一台机器可能会暂时停机，而您不想在其上部署软件。现在，您可以禁用它，而不是删除它或将其移动到一个虚假的环境中:

![Disabling a machine](img/84972e13e4a252835905e499744cd321.png)

禁用的计算机不会用于部署，也不会用于运行状况检查/升级。

愉快的部署！