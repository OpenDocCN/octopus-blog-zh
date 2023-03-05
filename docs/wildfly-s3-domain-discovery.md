# 野火 S3 域名发现-章鱼部署

> 原文：<https://octopus.com/blog/wildfly-s3-domain-discovery>

在云环境中配置 WildFly 域时，通常不可能依靠固定的 IP 地址或网络广播来发现资源。为 AWS 用户发现域控制器提供的一个解决方案是使用 S3 桶作为中央配置点。在这篇博文中，我们将看看如何在 AWS 中配置一个简单的 WildFly 域，使用 S3 桶来发现域控制器。

## AWS IAM 用户

首先，我们需要一个 IAM 用户，他有权限访问 S3 存储桶。WildFly 将使用该用户的密钥来用域控制器的详细信息更新 S3 桶。

在这个例子中，我们将使用一个名为`wildfly-domain`的桶。为了授予对这个 bucket 的访问权，WildFly 将使用其密钥的 IAM 用户具有以下安全策略。

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "Stmt1496702015000",
            "Effect": "Allow",
            "Action": [
                "s3:*"
            ],
            "Resource": [
                "arn:aws:s3:::wildfly-domain",
                "arn:aws:s3:::wildfly-domain/*"
            ]
        }
    ]
} 
```

有了这个策略，我们需要创建[访问键](http://docs.aws.amazon.com/IAM/latest/UserGuide/id_credentials_access-keys.html)。生成`Access key ID`和`Secret access key`时记下它们，因为 AWS 不会再向您显示秘密访问密钥。

## 创建存储桶

S3 水桶本身没什么特别的。它不需要启用任何特殊特性，因此只需创建一个名为安全策略中引用的名称的 bucket(在我们的例子中为`wildfly-domain`)。

## 配置域控制器

要配置域控制器以允许从属实例通过 S3 存储桶发现它并进行连接，需要很多步骤。

### 添加从属用户

我们需要做的第一件事是在域控制器上创建一个用户，从属实例将使用它来连接到域控制器。这是通过`bin/add-user.sh`脚本完成的。用户:

*   是一个`Management User`
*   不属于任何团体
*   将用于一个 AS 流程连接到另一个 AS 流程

创建名为`slave`的用户的输出如下所示。

```
[ec2-user@ip-172-30-0-89 bin]$ ./add-user.sh

What type of user do you wish to add?
 a) Management User (mgmt-users.properties)
 b) Application User (application-users.properties)
(a):

Enter the details of the new user to add.
Using realm 'ManagementRealm' as discovered from the existing property files.
Username : slave
Password recommendations are listed below. To modify these restrictions edit the add-user.properties configuration file.
 - The password should be different from the username
 - The password should not be one of the following restricted values {root, admin, administrator}
 - The password should contain at least 8 characters, 1 alphabetic character(s), 1 digit(s), 1 non-alphanumeric symbol(s)
Password :
Re-enter Password :
What groups do you want this user to belong to? (Please enter a comma separated list, or leave blank for none)[  ]:
About to add user 'slave' for realm 'ManagementRealm'
Is this correct yes/no? yes
Added user 'slave' to file '/home/ec2-user/wildfly-11.0.0.Final/standalone/configuration/mgmt-users.properties'
Added user 'slave' to file '/home/ec2-user/wildfly-11.0.0.Final/domain/configuration/mgmt-users.properties'
Added user 'slave' with groups  to file '/home/ec2-user/wildfly-11.0.0.Final/standalone/configuration/mgmt-groups.properties'
Added user 'slave' with groups  to file '/home/ec2-user/wildfly-11.0.0.Final/domain/configuration/mgmt-groups.properties'
Is this new user going to be used for one AS process to connect to another AS process?
e.g. for a slave host controller connecting to the master or for a Remoting connection for server to server EJB calls.
yes/no? yes
To represent the user add the following to the server-identities definition <secret value="UGFzc3dvcmQwMSE=" /> 
```

记下这条线

```
To represent the user add the following to the server-identities definition <secret value="UGFzc3dvcmQwMSE=" /> 
```

稍后我们将需要它来配置从属实例。

### 配置 S3 存储桶

在`domain/configuration/host-master.xml`文件中，用下面的代码替换`<domain-controller>`元素的默认内容。该配置指示域控制器在启动时将其详细信息保存在`wildfly-domain` S3 桶中。

```
<domain-controller>
  <local>
      <discovery-options>
          <discovery-option name="s3-discovery" code="org.jboss.as.host.controller.discovery.S3Discovery" module="org.jboss.as.host-controller">
            <property name="access-key" value="AKIAINKG7EYTNEPL2TBA"/>
            <property name="secret-access-key" value="yoursecretkeygoeshere"/>
            <property name="location" value="wildfly-domain"/>
        </discovery-option>
    </discovery-options>
  </local>
</domain-controller> 
```

### 运行域控制器

默认情况下，管理接口绑定到 localhost，在这种情况下，域控制器将尽职尽责地将配置保存在 S3 存储桶中，指示它可以在 127.0.0.1 上访问，当从属服务器在另一个实例上运行时，这显然不是有用的信息。

要解决这个问题，域控制器必须将其管理端口绑定到外部接口。这里，我们用`-bmanagement=<ip>`参数启动了域控制器，它指示 WildFly 将管理端口绑定到外部 NIC 的 IP 地址。

我们还通过`--host-config`参数提供了主机配置文件的名称。

```
[ec2-user@ip-172-30-0-89 bin]$ ./domain.sh --host-config=host-master.xml -bmanagement=172.30.0.89 
```

### 验证 S3 配置

一旦域控制器已经启动，你会发现新的文件创建在 S3 桶。

[![s3 domain config](img/340298689e589f353070abf5e9df0f18.png)](#)

## 配置域从属服务器

要配置从机，我们需要编辑`domain/configuration/host-slave.xml`文件。

用创建`slave`用户时由`add-users.sh`脚本生成的`<secret>`替换`<server-identities>`元素的内容。

```
<server-identities>
  <!-- Replace this with either a base64 password of your own, or use a vault with a vault expression -->
  <secret value="UGFzc3dvcmQwMSE=" />
</server-identities> 
```

然后用下面的代码替换`<domain-controller>`元素的内容。

```
<domain-controller>
  <remote security-realm="ManagementRealm" username="slave">
    <discovery-options>
        <discovery-option name="s3-discovery" code="org.jboss.as.host.controller.discovery.S3Discovery" module="org.jboss.as.host-controller">
          <property name="access-key" value="AKIAINKG7EYTNEPL2TBA"/>
          <property name="secret-access-key" value="yoursecretkeygoeshere"/>
          <property name="location" value="wildfly-domain"/>
        </discovery-option>
    </discovery-options>
  </remote>
</domain-controller> 
```

然后，可以使用命令启动从属实例

```
./domain.sh --host-config host-slave.xml 
```

## 在管理控制台中验证域

当域控制器和从控制器都在运行时，我们可以使用管理控制台来查看域的详细信息。打开 http://domaincontrollerip:9990。系统将提示您输入凭据。通常我们会为管理控制台创建一个专用用户，但是为了方便起见，我们可以只使用我们之前创建的`slave`用户。

在`Runtime`选项卡下，我们可以看到主主机和从主机确实按照预期连接。

[![wildfly domain management](img/ec1c0c6f2bb1e7be3c9b63d5cccfc713.png)](#)

## 结论

使用 S3 桶作为配置的中心点是在 AWS 中配置 WildFly 域的一种简单、方便和可靠的方式。它消除了对固定 IP 地址的需求，并解决了 AWS 阻止网络广播的限制。

如果您对 Java 应用程序的自动化部署感兴趣，[下载 Octopus Deploy 的试用版](https://octopus.com/downloads)，并查看我们的文档。