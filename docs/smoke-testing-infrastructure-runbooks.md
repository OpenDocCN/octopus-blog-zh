# 使用 runbooks - Octopus Deploy 对您的基础设施进行冒烟测试

> 原文：<https://octopus.com/blog/smoke-testing-infrastructure-runbooks>

*互联网坏了！*

任何在帮助台呆过一段时间的人都听过这种说法，以及其他类似的对客户遇到的问题的模糊描述。在诊断问题时，获得可操作的信息是成功的一半。

然而，当支持复杂的基础设施时，可能很难知道系统是如何设计的，这使得很难知道要问什么问题以及在哪里可以找到帮助解决问题的信息。这是企业环境中常见的自定义应用程序的本质，每个应用程序都是上一个应用程序的发展，或者每次都由完全不同的团队使用独特的方法编写。这意味着关于如何支持应用程序的业务知识通常只存在于少数员工的头脑中。

Runbooks 提供了一种以自动化和可测试的方式获取这些业务知识的方法，确保支持团队能够快速诊断高级问题，并有效地响应客户请求。

在这篇文章中，我提供了一个针对 1 级支持团队的示例操作手册，旨在对 AWS 中的典型微服务应用程序进行冒烟测试。

## 先决条件

这篇文章假设 runbook 步骤是在一个 Linux Worker 上运行的。他们使用 [dig](https://linux.die.net/man/1/dig) 进行 DNS 查找，使用 [hey](https://github.com/rakyll/hey) 进行负载测试，使用 [curl](https://curl.se/docs/projdocs.html) 与 HTTP 端点交互，使用 [mysql 客户端](https://dev.mysql.com/doc/refman/8.0/en/mysql.html)。

要在 Ubuntu 中安装这些工具，请运行以下命令:

```
apt-get install curl dnsutils mysql 
```

要在 Fedora、RHEL、Centos 或 Amazon Linux 中安装工具，请运行:

```
yum install curl mysql bind-utils 
```

然后用命令手动下载`hey`:

```
curl -o /usr/local/bin/hey https://hey-release.s3.us-east-2.amazonaws.com/hey_linux_amd64
chmod +x /usr/local/bin/hey 
```

我们创建了一个[公共 Octopus 实例，这个 runbook 是针对一个活动的微服务](https://tenpillars.octopus.app/app#/Spaces-42/projects/audits-service/operations/runbooks/Runbooks-102/overview)定义的。使用来宾帐户登录，查看操作手册步骤并列出以前执行的结果。

## 烟雾测试 DNS

DNS 让你把友好的名字，比如`development.octopus.pub`，映射到 IP 地址，比如`52.52.151.20`。

DNS 通常是一个稳定的服务，但当它失败时，很可能没有其他网络服务将正常工作。因此，您希望对暴露于 internet 的服务执行的第一个冒烟测试是验证 DNS 名称可以被解析。

下面的脚本执行`dig`来检查与域名相关的 DNS 记录:

```
dig "#{Octopus.Environment.Name | ToLower}.octopus.pub"
# Capture the return code of the previous command. This will be used as the exit code of the entire script once we print out
# any further instructions.
RETURN_CODE=$?
echo "============================"
echo "HOW TO INTERPRET THE RESULTS"
echo "============================"
echo "The dig command returns a lot of technical details, most of which are not important from a level 1 support point of view."
echo "As long as the command passes, you can assume the DNS is correctly configured."
echo "If the command fails, escalate this issue to level 2 support."
# Exit the script with the return code from the smoke test
exit $RETURN_CODE 
```

该脚本的输出如下所示:

[![dig output](img/8c7123458ee35d0c4557470aabff5fec.png)](#)

像`dig`这样的工具往往技术性很强，输出需要一些经验来解释。然而，您的 1 级支持团队通常不需要深入了解 DNS 等网络问题，因此脚本必须解释结果和任何进一步的行动。这是在操作手册中获取商业知识的一个例子，这意味着即使是新手也可以运行这些操作手册，并有信心对结果做出反应。

## 冒烟测试 MySQL

在这个例子中，我们的应用程序使用 MySQL 数据库来实现持久性。如果数据库不可访问，服务将失败，因此下一步是编写一个脚本对数据库进行冒烟测试。

下面的脚本使用`mysql`命令行工具来尝试查询一个已知的数据库表。注意，结果被重定向到`/dev/null`，因为我们不想用实际的数据库记录填充日志:

```
DATABASE_HOST=$(get_octopusvariable "Database.Host")
USERNAME=$(get_octopusvariable "Database.AuditsUsername")
PASSWORD=$(get_octopusvariable "Database.AuditsPassword")

echo "Database Host: $DATABASE_HOST"
echo "Database Username: $USERNAME"

mysql --host=$DATABASE_HOST --user=$USERNAME --password=$PASSWORD audits -e "SELECT * FROM audits" > /dev/null
# Capture the return code of the previous command. This will be used as the exit code of the entire script once we print out
# any further instructions.
RETURN_CODE=$?

echo "============================"
echo "HOW TO INTERPRET THE RESULTS"
echo "============================"
echo "This test attempts to query the audits database."
echo "If this step fails, escalate the issue to level 2 support."

# Exit the script with the return code from the smoke test
exit $RETURN_CODE 
```

## 冒烟测试 HTTP 服务

下一个测试验证公共 HTTP 端点是否响应了预期的状态代码。Web 服务总是随任何响应返回一个状态代码，通常您可以假设一个公共 URL 将返回一个代码 200，这表示响应成功。

有关 HTTP 响应代码的完整列表，请参考 [MDN 文档](https://developer.mozilla.org/en-US/docs/Web/HTTP/Status)。

对于这个测试，我们利用了[社区步骤模板库](https://octopus.com/docs/projects/community-step-templates)中一个名为 **HTTP - Test URL (Bash)** 的步骤。这一步定义了一个 Bash 脚本，它根据提供的 URL 调用`curl`,并验证 HTTP 状态代码:

[![HTTP - Test URL (Bash)](img/7bdd6786d77a0ce1380c11600d800ace.png)](#)

## 负载测试

前面的三次冒烟测试验证了我们的应用程序基础设施的基本层。你可以预期，如果他们中的任何一个失败了，就会出现严重的问题。

然而，应用程序仍有可能工作，但由于速度慢或随机请求失败而不可用。您的最终冒烟测试使用`hey`执行快速负载测试，以验证应用程序对多个请求的响应是否一致。下面的脚本针对微服务 API 调用`hey`:

```
# Warm up
hey https://#{Octopus.Environment.Name | ToLower}.octopus.pub/api/audits > /dev/null

# Real test
hey https://#{Octopus.Environment.Name | ToLower}.octopus.pub/api/audits
# Capture the return code of the previous command. This will be used as the exit code of the entire script once we print out
# any further instructions.
RETURN_CODE=$?

echo "============================"
echo "HOW TO INTERPRET THE RESULTS"
echo "============================"
echo "It is expected that the majority of requests complete in under a second."
echo "If the chart above shows the majority of requests taking longer than a second, please escalate this issue to level 2 support."
# Exit the script with the return code from the smoke test
exit $RETURN_CODE 
```

该脚本的输出显示在下面的屏幕截图中:

[![hey output](img/bfabdb4c4373f26ef94e4554d33117a2.png)](#)

这个输出需要一些解释来决定采取什么进一步的行动。直方图显示了每个请求的响应时间，在本例中，您预期大多数请求在不到一秒钟的时间内完成。这些说明指导运行该脚本的团队成员根据输出做出适当的决策。

## 结论

您在企业环境中遇到的每个应用程序都需要大量底层服务和基础设施才能正常运行。通过编写探测和验证这些层的冒烟测试，支持团队可以快速确认问题，并高效、自信地响应支持请求。

在本文中，您看到了验证 DNS 服务、HTTP 端点和 MySQL 数据库的冒烟测试示例。您还看到了一个简单的负载测试，它提供了对服务响应多个请求时的性能的洞察。

阅读我们的 [Runbooks 系列](https://octopus.com/blog/tag/Runbooks%20Series)的其余部分。

愉快的部署！