# 将存储库部署到 WildFly - Octopus 部署

> 原文：<https://octopus.com/blog/wildfly-vault>

在处理密码等敏感信息时， [Octopus Deploy 为您提供加密和保存这些值的能力](https://octopus.com/docs/projects/variables/sensitive-variables)以确保它们的安全。

如果这些密码是用于数据库服务器之类的外部系统，通常需要解密密码，并将其以纯文本形式存储在某个配置文件中，以便实际使用密码。但是以纯文本的形式保存敏感信息并不理想，因为这使得它容易受到许多简单的攻击，例如在编辑配置文件时监视他人，或者在聊天系统中共享、通过电子邮件发送或发布到帮助论坛，或者检查版本控制系统的历史时捕捉内容文件的内容。

WildFly 通过将敏感信息放入保险库中来缓解这些漏洞。

虽然保险库被认为是安全的，但将敏感信息放入保险库仍然是值得的，因为这意味着配置文件的普通观察者将无法提取密码。

作为为 Java 提供一流支持的举措的一部分，我们正计划提供将 Octopus 变量导出到 WildFly vault 的能力。在这篇博文中，我将向你介绍如何在今天实现这一目标。

## 导出变量

如果您曾经使用过 WildFly 附带的`vault`脚本，您会知道要存储在 vault 中的每个值都必须一次添加一个。这很乏味，而且很难维护。为了提供一种更快的方式将安全值放入保险库中，我们编写了一个 [Groovy 脚本](https://github.com/OctopusDeploy/JBossDeployment/blob/master/create-vault.groovy)，它从一个 CSV 文件中获取值。

但是首先我们需要从 Octopus 中获取安全值并保存到一个 CSV 文件中。幸运的是，Powershell 附带了`Export-Csv`命令，这使得事情变得微不足道。下面的代码可以在 Octopus 的脚本步骤中定义，以将一组键/值对导出到 CSV 文件。然后它调用`create-vault.groovy`脚本将 CSV 文件转换成 WildFly vault。保险库密码保存在名为`VaultPassword`的输出变量中，CSV 文件被删除。

```
try {
    # Dump all available variables
    # $OctopusParameters.GetEnumerator() | sort-object Name  | Export-Csv -Path "C:\variables.csv"

    # Dump only a few variables
    $variables = @{"wildfly_slave_password" = $wildfly_slave_password}
    $variables.GetEnumerator() | Select Key,Value  | Export-Csv -Path "C:\variables.csv"

    cd C:\Apps\JBossDeployment
    $vaultPassword = &groovy create-vault.groovy `
      --keystore-file C:\wildfly_dc\wildfly-11.0.0.Alpha1\domain\configuration\keystore.vault `
      --keystore-password Password01 `
      --enc-dir C:\wildfly_dc\wildfly-11.0.0.Alpha1\domain\configuration\vault `
      --csv C:\variables.csv

    Set-OctopusVariable -name "VaultPassword" -value $vaultPassword
}
finally {
    rm "C:\variables.csv"
} 
```

一旦这个脚本运行，我们将有一个名为`C:\wildfly_dc\wildfly-11.0.0.Alpha1\domain\configuration\keystore.vault`的文件，一个名为`C:\wildfly_dc\wildfly-11.0.0.Alpha1\domain\configuration\vault`的目录，以及一个可以用来访问保险库的密码。

您需要首先在所有域节点或独立节点上运行此脚本，以便它们都有一个 vault 的本地副本，可以在下一步中进行配置。

## 配置存储库

创建了保险库之后，我们现在需要配置 WildFly 来使用它。这是通过将以下 XML 添加到 WildFly 主机配置文件中来完成的。

```
<vault>
    <vault-option name="KEYSTORE_URL" value="C:\wildfly_standalone\wildfly-11.0.0.Alpha1\standalone\configuration\keystore.vault"/>
    <vault-option name="KEYSTORE_PASSWORD" value="MASK-223/wEo1GLELe8EuQa5u20"/>
    <vault-option name="KEYSTORE_ALIAS" value="vault"/>
    <vault-option name="SALT" value="12345678"/>
    <vault-option name="ITERATION_COUNT" value="50"/>
    <vault-option name="ENC_FILE_DIR" value="C:\wildfly_standalone\wildfly-11.0.0.Alpha1\standalone\configuration\vault\/"/>
</vault> 
```

为了简化这个过程，有另一个 [Groovy 脚本](https://github.com/OctopusDeploy/JBossDeployment/blob/master/deploy-vault.groovy)将为您添加这个配置。

下面的 Powershell 用于运行 Groovy 脚本并将 vault 配置添加到主机。

您只需要在域控制器上运行这个脚本，而不需要在域从属服务器上运行，因为域控制器会将 vault 配置推送到从属服务器。但是，域控制器不会推出实际的 vault 文件，这就是我们在添加此配置之前在所有主机上创建 vault 文件的原因。

如果您有独立的实例，那么这个脚本将在所有的实例上运行。

```
cd C:\Apps\JBossDeployment
&groovy deploy-vault.groovy `
  --controller localhost `
  --port 9990 `
  --user admin `
  --password password `
  --keystore-file C:\wildfly_dc\wildfly-11.0.0.Alpha1\domain\configuration\keystore.vault `
  --keystore-password $OctopusParameters["Octopus.Action[Create Vault].Output.VaultPassword"] `
  --enc-dir C:\wildfly_dc\wildfly-11.0.0.Alpha1\domain\configuration\vault 
```

## 访问保险库密码

通过 WildFly 配置文件中的一个特殊变量，可以访问存储库中包含的密码。这里，我们引用了我们的 vault 中别名`vault`的`wildfly_slave_password`变量。

```
<server-identities>
  <secret value="${VAULT::vault::wildfly_slave_password::1}"/>
</server-identities> 
```

正如您所看到的，实际的密码不再以纯文本的形式保存，即使这个配置文件被泄露，它也不会泄露一个泄漏的密码。

## 后续步骤

这些 Groovy 脚本正在被开发，作为最终将被移植到 Octopus Deploy 中直接提供的步骤中的概念验证。

如果你对剧本有任何问题，请留下评论。如果有一些 Java 特性你希望 Octopus 在未来部署支持，请加入 [Java RFC 帖子](https://octopus.com/blog/java-rfc)的讨论。