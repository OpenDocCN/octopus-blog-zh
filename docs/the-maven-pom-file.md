# Selenium 系列:Maven POM 文件- Octopus 部署

> 原文：<https://octopus.com/blog/selenium/2-the-maven-pom-file/the-maven-pom-file>

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。

建立我们的 Java 项目的第一步是创建一个 Maven 项目对象模型(POM)文件。这是一个 XML 文档，它定义了我们的代码将如何构建，它可以访问哪些附加的依赖项，以及测试如何运行。

我们从下面显示的 POM 文件开始:

```
<project 
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <groupId>com.octopus</groupId>
  <artifactId>webdrivertraining</artifactId>
  <version>1.0-SNAPSHOT</version>
  <packaging>jar</packaging>
  <properties>
    <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
    <java.version>1.8</java.version>
    <selenium.version>3.14.0</selenium.version>
    <junit.version>4.12</junit.version>
  </properties>
  <build>
    <plugins>
      <plugin>
        <groupId>org.apache.maven.plugins</groupId>
        <artifactId>maven-compiler-plugin</artifactId>
        <version>3.1</version>
        <configuration>
          <source>${java.version}</source>
          <target>${java.version}</target>
        </configuration>
      </plugin>
    </plugins>
  </build>
  <dependencies>
    <dependency>
      <groupId>org.seleniumhq.selenium</groupId>
      <artifactId>selenium-java</artifactId>
      <version>${selenium.version}</version>
    </dependency>
    <dependency>
      <groupId>junit</groupId>
      <artifactId>junit</artifactId>
      <version>${junit.version}</version>
      <scope>test</scope>
    </dependency>
  </dependencies>
</project> 
```

让我们分解这个 POM 文件来理解各个组件。

## 项目元素

顶层是`<project>`元素:

```
<project 
xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
http://maven.apache.org/xsd/maven-4.0.0.xsd">
  <modelVersion>4.0.0</modelVersion>
  <!-- ... -->
</project> 
```

所有 Maven POM 文件都有一个`<project>`元素作为根元素，属性定义了 XML 名称空间和 Maven XML 模式的细节。

元素定义了 POM 版本。该元素唯一支持的值是`4.0.0`。

[https://maven.apache.org/pom.html#Quick_Overview](https://maven.apache.org/pom.html#Quick_Overview)详细介绍了组成 POM 文件的元素。

## 组、工件和版本

`<groupId>`、`<artifactId>`和`<version>`元素定义了这个项目产生的 Maven 工件的身份。这些值有时被组合起来，缩写为 GAV。

`groupId`通常采用反向域名的形式，尽管这只是惯例，并不是严格的要求。我们使用了值`com.octopus`，它是 URL【https://octopus.com/】的域名[的反义词。](https://octopus.com/)

`groupId`和`artifactId`的组合必须是唯一的，因为许多项目可能共享`groupId`，所以用`artifactId`来描述这个项目。

如您所料，`version`元素定义了这个库的版本。`1.0-SNAPSHOT`的值意味着该代码正在向 1.0 版本努力，但是还没有达到稳定:

```
<groupId>com.octopus</groupId>
<artifactId>webdrivertraining</artifactId>
<version>1.0-SNAPSHOT</version> 
```

## 包装

元素定义了 Maven 将产生的工件的类型。在我们的例子中，我们正在生成一个 JAR 文件:

```
<packaging>jar</packaging> 
```

## 性能

`<properties>`元素为属性定义了值，这些值可以被 Maven 直接识别，也可以作为共享公共值的一种方式在 POM 文件中被引用。

这里我们已经设置了`project.build.sourceEncoding`属性，这是一个由 Maven 识别的设置，它定义了我们的代码所依赖的依赖项的版本，并在`java.version`属性中定义了我们的代码将要编译的 Java 版本:

```
<properties>
  <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
  <java.version>1.8</java.version>
  <selenium.version>3.12.0</selenium.version>
  <junit.version>4.12</junit.version>
</properties> 
```

属性`project.build.sourceEncoding`被设置为`UTF-8`以指定跨操作系统的 Java 源文件的一致编码。如果未设置该值，不同的操作系统会认为 Java 文件具有不同的编码，这可能会在代码由不同的人或服务编译时导致问题。在后面的文章中，我们将配置一个基于 Linux 的外部服务来为我们构建代码，设置这个属性意味着我们的构建将按预期工作，不管我们在本地使用什么操作系统开发代码。

## 插件

接下来的设置配置 Maven 构建过程的某些方面。Maven 项目有一个众所周知的默认生命周期，它包含以下默认阶段，如下图所示。

[![](img/313df6b6ea6532fab6034d1610995795.png)](#)

*   clean -删除上一次构建生成的所有文件。
*   验证-验证项目是正确的，并且所有必要的信息都是可用的。
*   编译-编译项目的源代码。
*   使用合适的单元测试框架测试编译后的源代码。这些测试不需要打包或部署代码。
*   打包——将编译好的代码打包成可分发的格式，比如 JAR。
*   验证-对集成测试的结果进行检查，以确保符合质量标准。
*   install——将包安装到本地存储库中，作为本地部署的其他项目的依赖项。
*   站点-生成项目的站点文档。
*   部署——在集成或发布环境中完成，将最终的包复制到远程存储库中，以便与其他开发人员和项目共享。

请参见[https://maven . Apache . org/guides/introduction/introduction-to-the-life cycle . html # life cycle _ Reference](https://maven.apache.org/guides/introduction/introduction-to-the-lifecycle.html#Lifecycle_Reference)了解 Maven 生命周期阶段的完整列表。

每个阶段都由一个插件公开的目标来执行，每个插件都可以在嵌套在`<build><plugins>`元素下的`<plugin>`元素中配置。

对于我们的项目，我们需要配置`maven-compiler-plugin`，它用于在编译阶段编译我们的 Java 代码，以指示它构建与 Java 1.8 兼容的工件。

Java 版本的配置是通过`<source>`和`<target>`元素完成的。这些元素的值都被设置为`${java.version}`，这就是我们引用前面在`<properties>`元素中定义的名为`java.version`的自定义属性的值的方式:

```
<build>
  <plugins>
    <plugin>
      <groupId>org.apache.maven.plugins</groupId>
      <artifactId>maven-compiler-plugin</artifactId>
      <version>3.1</version>
      <configuration>
        <source>${java.version}</source>
        <target>${java.version}</target>
      </configuration>
    </plugin>
  </plugins>
</build> 
```

## 属国

最后，我们有了`<dependencies>`元素。这个元素定义了我们的应用程序所依赖的附加库。

依赖关系的细节通常可以通过许多 Maven 知识库搜索引擎中的一个找到，比如[https://search.maven.org/](https://search.maven.org/)。下面的屏幕截图显示了对 JUnit 库的搜索:

[![](img/859c5538cf986e539713c53b3a63cf5b.png)](#)

结果包括具有许多不同 GroupIDs 的工件，但是我们想要的工件具有`GroupID`和`junit`的`ArtifactID`。结果还告诉我们最新的版本是什么。

[![](img/bdded97d962b592dc4cfefeeb8788de4.png)](#)

点击版本号(上面例子中的 4.12 链接)会把你带到下面的页面，这个页面提供了可以添加到一个`pom.xml`文件中的 XML，以包含这个依赖项。

[![](img/5de5221ed94d8e322d342995a9c3d1d7.png)](#)

JUnit 是一个流行的单元测试库，我们将在编写 WebDriver 测试时广泛使用它。此外，我们添加了对`selenium-java`库的依赖，它提供了 WebDriver API 的实现。

注意`<scope>`元素被设置为`test`用于`junit`依赖关系。这表明这种依赖性只在 Maven 生命周期的测试阶段需要，在构建发布工件时不会被打包。

我们在这里使用的是 JUnit 4，而不是更新的 JUnit 5，因为我们将在后面的课程中使用的一些库还没有更新到可以与 JUnit 5 一起使用。

```
<dependencies>
  <dependency>
    <groupId>org.seleniumhq.selenium</groupId>
    <artifactId>selenium-java</artifactId>
    <version>${selenium.version}</version>
  </dependency>
  <dependency>
    <groupId>junit</groupId>
    <artifactId>junit</artifactId>
    <version>${junit.version}</version>
    <scope>test</scope>
  </dependency>
</dependencies> 
```

将`pom.xml`文件保存到本地 PC 上的某个位置。一旦文件被保存，我们可以将它导入 IntelliJ，从这里我们将继续编写 Java 代码。

首次打开 IntelliJ 时，可以选择从初始对话框中导入项目。单击导入项目链接。

[![](img/e6e70fbc73a0e0f71f6a179b72c9fb29.png)](#)

然后选择`pom.xml`文件。

[![](img/dcc4b330edd3226ebad4171f30270d8c.png)](#)

IntelliJ 识别出 pom.xml 文件是一个 Maven 项目，并开始从 Maven 导入项目。

这个对话框中的默认值是好的，所以点击`Next`按钮。

[![](img/3fc9d761ee1717f4988a64216578a977.png)](#)

自动检测`com.octopus:webdrivertraining:1.0-SNAPSHOT`(这是组、工件和版本，或者 GAV，工件的表示)项目。

点击`Next`按钮。

[![](img/1003fd24aaedbd92fe05a6c0bd37ffd0.png)](#)

此对话框允许您选择用于构建项目的 JDK。如果您是第一次安装 IntelliJ，此列表将为空。

[![](img/d50158b3546fcee74d6955bb2730a730.png)](#)

如果您没有看到 Java 1.8 SDK 选项，请单击加号图标并从`Add New SDK`菜单中选择 JDK。

[![](img/ee16b7410b79b1ea04812a3606195ca9.png)](#)

选择 Java 1.8 JDK 的安装目录，点击`Open`或`OK`按钮。

在 MacOS 系统上，Oracle JDK 通常位于类似`/Library/Java/JavaVirtualMachines/jdk1.8.0_144.jdk/Contents/Home`的目录下。

[![](img/742bfa7d4d1fd2694e0dfd1264015e8b.png)](#)

在 Windows 系统中，你通常会在类似于`C:\Program Files\Java\jdk1.8.0_144`的目录下找到 JDK。

[![](img/612049ee712688e72fcbc4b8681fc610.png)](#)

在像 Ubuntu 这样的 Linux 系统上，JDK 可以在像`/usr/lib/jvm/java-8-openjdk-amd64`这样的目录下找到。

[![](img/28e46c92241e7c51b3dafb5569ca552d.png)](#)

这将把 JDK 添加到列表中。点击`Next`按钮。

[![](img/f1bf15087f59d6b938db1d11dcac86d1.png)](#)

IntelliJ 项目的默认名称与 Maven ArtifactID 相同。这是默认设置，所以点击`Finish`按钮。

[![](img/8db9de1744bdbd8a8b4cf9962851c228.png)](#)

然后 IntelliJ 创建一个链接到 Maven 项目的空项目。

在 ide 的右边你会看到一个叫做`Maven Projects`的按钮。如果看不到这个按钮，可以通过点击查看➜工具窗口➜ Maven 项目打开窗口。

[![](img/1070045cec4d7a32d3bc77c4073f2ed2.png)](#)

使用`View`菜单或按钮打开该菜单，将显示`Maven Projects`窗口。在`Lifecycle`菜单项下，您可以看到构成默认生命周期的 Maven 阶段。

[![](img/8cb195672d08c4e591cda63112c72184.png)](#)

双击阶段用 Maven 执行它们。这里我们已经执行了`package`阶段。这在目前不会取得很大的成就，因为我们没有要编译或打包的代码，但是操作成功的事实(我们可以从输出中的`BUILD SUCCESS`消息中看到)意味着我们有一个有效的`pom.xml`文件。

[![](img/fde8023ffb8e534eadb7f55e86deaf28.png)](#)

在我们结束这篇文章之前，我想演示一下设置`project.build.sourceEncoding`属性的效果。从`pom.xml`文件中注释掉这一行并保存修改。

IntelliJ 将检测到文件已被修改，并为您提供自动导入任何未来更改的选项。这通常是个好主意，所以单击通知弹出窗口中的`Enable Auto-Import`链接。

[![](img/699c7bd6ed11bb2da7729d1cc4e5f6f2.png)](#)

从`Maven Projects`工具窗口再次双击封装阶段。

请注意，当没有设置`project.build.sourceEncoding`属性时，在构建项目时，Maven 会显示以下警告:

```
[WARNING] Using platform encoding (UTF-8 actually) to copy filtered resources, i.e. build is platform dependent! 
```

恢复变更并再次打包项目，您会注意到警告消失了。

[![](img/211fe774c4f40137a2dca53db3e5ae4d.png)](#)

这样我们就有了一个导入 IntelliJ 的最小 Maven 项目。在下一篇文章中，我们将开始编写一些 Java 来实现我们的第一个 WebDriver 测试。

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。