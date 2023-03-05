# 如何处理锁定的文件和部署-八达通部署

> 原文：<https://octopus.com/blog/how-to-handle-locked-files>

锁定的文件是否阻碍了您的部署？您是否尝试过在 PowerShell 步骤中关闭网站，但仍然出现锁定文件错误？你不是唯一一个。

这是我们收到的非常常见的支持请求。错误消息，例如:

> 无法将包复制到指定的目录“c:\MyDeploymentLocation”。目录中的一个或多个文件可能被另一个进程锁定。

可能会令人沮丧，并且试图通过停止您认为可能正在访问文件的进程来找到锁的来源有点像猜谜游戏。

## 引入手柄

![SysInternals](img/c7f03a8c678ca379cf09b39b1598b623.png)

猜不到了！句柄是一个来自 SysInternals 的工具，用来显示所有打开文件的程序(并因此被锁定！)

如果您查看您的特定错误并找到文件名:

```
The process cannot access the file 'c:\MyDeploymentLocation\Web.config' 
```

然后，您可以使用句柄来查找打开文件内容，例如:

```
C:\Handle> handle C:\MyDeploymentLocation

Handle v3.51
Copyright (C) 1997-2013 Mark Russinovich
Sysinternals - www.sysinternals.com

explorer.exe       pid: 3732   type: File           C7C: C:\MyDeploymentLocation\Web.config 
```

这将允许您使用一个 [PowerShell](http://docs.octopusdeploy.com/display/OD/PowerShell+scripts) 脚本步骤来首先停止这些进程，允许您在没有锁定文件妨碍的情况下完成部署！

**更新:**

当 Handle.exe 第一次运行时，系统会提示您接受最终用户许可协议。接受协议存储在每个用户的注册表中。如果你从 Octopus Deploy 触手代理运行 Handle.exe，它挂起了，这很可能是因为触手是在一个没有接受许可协议的用户帐户下运行的。您可以通过在调用 Handle.exe 之前运行以下命令来自动接受许可协议:

```
& reg.exe ADD "HKCU\Software\Sysinternals\Handle" /v EulaAccepted /t REG_DWORD /d 1 /f 
```