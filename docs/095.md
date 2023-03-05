# 构建 Apache 可移植运行时(APR) - Octopus 部署

> 原文：<https://octopus.com/blog/building-apr-for-tomcat>

Tomcat 使用 Apache Portable Runtime (APR) 来提供许多增强的特性和性能。例如，为了获得 OpenSSL 为 HTTPS 提供的增强性能，APR 需要存在。

Tomcat 为 Windows 用户提供了 APR 的预编译版本，但是 Linux 用户经常被要求从他们的操作系统分发包库中安装 APR。

这些是可从 Centos 7 的 [EPEL 回购](https://fedoraproject.org/wiki/EPEL)获得的 APR 和 Tomcat 原生包:

```
$ yum info apr
Loaded plugins: fastestmirror, langpacks
Determining fastest mirrors
 * base: centos.mirror.ausnetservers.net.au
 * epel: fedora.melbourneitmirror.net
 * extras: mirror.nsw.coloau.com.au
 * updates: mirror.nsw.coloau.com.au
Installed Packages
Name        : apr
Arch        : x86_64
Version     : 1.4.8
Release     : 3.el7
Size        : 221 k
Repo        : installed
From repo   : anaconda
Summary     : Apache Portable Runtime library
URL         : http://apr.apache.org/
License     : ASL 2.0 and BSD with advertising and ISC and BSD
Description : The mission of the Apache Portable Runtime (APR) is to provide a
            : free library of C data structures and routines, forming a system
            : portability layer to as many operating systems as possible,
            : including Unices, MS Win32, BeOS and OS/2.

Available Packages
Name        : apr
Arch        : i686
Version     : 1.4.8
Release     : 3.el7
Size        : 107 k
Repo        : base/7/x86_64
Summary     : Apache Portable Runtime library
URL         : http://apr.apache.org/
License     : ASL 2.0 and BSD with advertising and ISC and BSD
Description : The mission of the Apache Portable Runtime (APR) is to provide a
            : free library of C data structures and routines, forming a system
            : portability layer to as many operating systems as possible,
            : including Unices, MS Win32, BeOS and OS/2.

$ yum info tomcat-native
Loaded plugins: fastestmirror, langpacks
Loading mirror speeds from cached hostfile
 * base: centos.mirror.ausnetservers.net.au
 * epel: fedora.melbourneitmirror.net
 * extras: mirror.nsw.coloau.com.au
 * updates: mirror.nsw.coloau.com.au
Installed Packages
Name        : tomcat-native
Arch        : x86_64
Version     : 1.1.34
Release     : 1.el7
Size        : 172 k
Repo        : installed
From repo   : epel
Summary     : Tomcat native library
URL         : http://tomcat.apache.org/tomcat-8.0-doc/apr.html
License     : ASL 2.0
Description : Tomcat can use the Apache Portable Runtime to provide superior
            : scalability, performance, and better integration with native server
            : technologies.  The Apache Portable Runtime is a highly portable library
            : that is at the heart of Apache HTTP Server 2.x.  APR has many uses,
            : including access to advanced IO functionality (such as sendfile, epoll
            : and OpenSSL), OS level functionality (random number generation, system
            : status, etc), and native process handling (shared memory, NT pipes and
            : Unix sockets).  This package contains the Tomcat native library which
            : provides support for using APR in Tomcat. 
```

所以我们可以访问提供 APR 版本 1.4.8 和 Tomcat 原生版本 1.1.34 的包。但是安装了这些软件包后，Tomcat 9.01 会报告以下错误:

```
org.apache.catalina.core.AprLifecycleListener.init An incompatible version [1.1.34] of the APR based Apache Tomcat Native library is installed, while Tomcat requires version [1.2.14] 
```

这意味着我们可以从 Centos 软件包管理器安装的版本对于 Tomcat 的更高版本来说不够新。

解决方法是自己编译这些库。这听起来很复杂，但实际上很简单。

首先，您需要安装开发工具和 Java 开发工具包。在 Centos 中，这是通过以下命令完成的:

```
sudo yum groupinstall "Development Tools"
sudo yum install java-1.8.0-openjdk-devel 
```

接下来运行下面的脚本，它将下载、提取、编译和安装 Tomcat 所需的各种库。

这个脚本中的 URL 指向[https://Tomcat . Apache . org/download-Native . CGI](Tomcat Native)库和 [APR](https://apr.apache.org/download.cgi) 库的镜像。您可以更改这些 URL 以指向一个更近的服务器，或者如果此处列出的服务器已关闭，则指向一个不同的服务器。

```
#!/usr/bin/env bash
cd /tmp
wget http://apache.mirror.amaze.com.au//apr/apr-1.6.3.tar.gz
tar -zxvf apr-1.6.3.tar.gz
cd apr-1.6.3
./configure
make
make install

cd /tmp
wget http://apache.mirror.amaze.com.au//apr/apr-util-1.6.1.tar.gz
tar -zxvf apr-util-1.6.1.tar.gz
cd apr-util-1.6.1
./configure --with-apr=/usr/local/apr
make
make install

cd /tmp
wget http://apache.melbourneitmirror.net/tomcat/tomcat-connectors/native/1.2.14/source/tomcat-native-1.2.14-src.tar.gz
tar -zxvf tomcat-native-1.2.14-src.tar.gz
cd tomcat-native-1.2.14-src/native
./configure --with-apr=/usr/local/apr --with-java-home=/usr/lib/jvm/java-1.8.0-openjdk
make
make install 
```

最后，在 Tomcat 目录下创建一个名为`bin/setenv.sh`的文件，内容如下。这允许 Tomcat 找到由上面的脚本编译的库。

```
CATALINA_OPTS="-Djava.library.path=/usr/local/apr/lib" 
```

完成这些更改后，您应该会在`logs/catalina.out`文件中看到以下消息。这表明 Tomcat 成功地加载了 APR 库。

```
31-Oct-2017 18:21:08.930 INFO [main] org.apache.catalina.core.AprLifecycleListener.lifecycleEvent Loaded APR based Apache Tomcat Native library [1.2.14] using APR version [1.6.3]. 
```

如果您对将 Java 应用程序自动部署到 Tomcat 感兴趣，[下载 Octopus Deploy](https://octopus.com/downloads) 的试用版，并查看一下[我们的文档](https://octopus.com/docs/deployments/java/deploying-java-applications)。