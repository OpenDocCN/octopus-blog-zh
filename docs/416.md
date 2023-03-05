# Maven 版本说明- Octopus 部署

> 原文：<https://octopus.com/blog/maven-versioning-explained>

[![Maven versions explained](img/8dda823895fcdf71604d5eba3502566b.png)](#)

版本字符串通常很容易理解，但是 Maven 有许多规则和边缘情况不是很明显。在这篇文章中，我将看看 Maven 版本的字符串是如何工作的。

## 真理的来源

Maven 发行版包括一个名为 [ComparableVersion](https://github.com/apache/maven/blob/master/maven-artifact/src/main/java/org/apache/maven/artifact/versioning/ComparableVersion.java) 的类，它是不同版本字符串如何相互比较的事实来源。

通过针对这个类编写一些测试，我们可以探索 Maven 版本是如何工作的。

## 版本的排序列表

让我们从一个测试开始，该测试获取一个由`ComparableVersion`对象组成的数组，克隆该数组，对其进行排序，并将其与原始列表进行比较。测试通过的事实证明了原始列表从最早版本到最新版本是有序的:

```
private static final ComparableVersion[] VERSIONS = new ComparableVersion[]{
        new ComparableVersion("NotAVersionSting"),
        new ComparableVersion("1.0a1-SNAPSHOT"),
        new ComparableVersion("1.0-alpha1"),
        new ComparableVersion("1.0beta1-SNAPSHOT"),
        new ComparableVersion("1.0-b2"),
        new ComparableVersion("1.0-beta3.SNAPSHOT"),
        new ComparableVersion("1.0-beta3"),
        new ComparableVersion("1.0-milestone1-SNAPSHOT"),
        new ComparableVersion("1.0-m2"),
        new ComparableVersion("1.0-rc1-SNAPSHOT"),
        new ComparableVersion("1.0-cr1"),
        new ComparableVersion("1.0-SNAPSHOT"),
        new ComparableVersion("1.0"),
        new ComparableVersion("1.0-sp"),
        new ComparableVersion("1.0-a"),
        new ComparableVersion("1.0-RELEASE"),
        new ComparableVersion("1.0-whatever"),
        new ComparableVersion("1.0.z"),
        new ComparableVersion("1.0.1"),
        new ComparableVersion("1.0.1.0.0.0.0.0.0.0.0.0.0.0.1")
};

@Test
public void ensureArrayInOrder() {
    ComparableVersion[] sortedArray = VERSIONS.clone();
    Arrays.sort(sortedArray);
    assertArrayEquals(VERSIONS, sortedArray);
} 
```

这个列表揭示了一些关于 Maven 版本之间相互比较的有趣事实。

限定词如`alpha`、`beta`、`milestone`(或它们的速记等价物`a`、`b`和`mc`)、`rc`、`sp`、`ga`和`final`有特殊的含义。句点和破折号等分隔符可以互换使用，或者在某些情况下根本不使用。不遵循任何特定格式的版本字符串仍然是有效的和可比较的。

## Maven 版本组件

虽然`ComparableVersion`类是版本相互比较的事实来源，但它并不解析特别有用的数据结构中的版本。为此，我们有来自[构建助手](http://www.mojohaus.org/build-helper-maven-plugin/parse-version-mojo.html)插件的第二个类，名为[版本信息](https://github.com/mojohaus/build-helper-maven-plugin/blob/master/src/main/java/org/codehaus/mojo/buildhelper/versioning/VersionInformation.java)。

`VersionInformation`将 Maven 版本字符串分解成 5 个部分:

*   重要的
*   较小的
*   修补
*   内部版本号
*   预选赛

主要、次要、修补和内部版本号都是整数值。

限定符可以包含任何值，尽管有些限定符确实有特殊的含义。

## 限定符和别名

Maven 可以识别许多特殊的限定符，这里按优先顺序列出:

*   阿尔法还是阿尔法
*   beta 或 b
*   里程碑或 m
*   rc 或 cr
*   快照
*   (空字符串)或 ga 或 final
*   sp

我们在排序版本列表中看到，这些限定符确实导致 Maven 版本按照与项目符号列表相同的顺序排序。

带有无法识别的限定符的版本被视为比非限定版本更晚的版本，无法识别的限定符被作为不区分大小写的字符串进行比较。

一些限定符有速记别名。该测试显示了各种限定符如何产生相同的 Maven 版本:

```
@Test
public void testAliases() {
    assertEquals(new ComparableVersion("1.0-alpha1"), new ComparableVersion("1.0-a1"));
    assertEquals(new ComparableVersion("1.0-beta1"), new ComparableVersion("1.0-b1"));
    assertEquals(new ComparableVersion("1.0-milestone1"), new ComparableVersion("1.0-m1"));
    assertEquals(new ComparableVersion("1.0-rc1"), new ComparableVersion("1.0-cr1"));
}

@Test
public void testDifferentFinalReleases() {
    assertEquals(new ComparableVersion("1.0-ga"), new ComparableVersion("1.0"));
    assertEquals(new ComparableVersion("1.0-final"), new ComparableVersion("1.0"));
} 
```

请注意，速记别名后面必须有一个数字，而它们的完全对等名称后面没有数字。如果你仔细看这篇文章开头介绍的排序版本列表，你会发现版本`1.0-alpha`和`1.0a1-SNAPSHOT`是最早的两个版本，而`1.0-a`在列表的末尾。

所有限定符都不区分大小写，如本测试所示:

```
@Test
public void testCase() {
    assertEquals(new ComparableVersion("1.0ALPHA1"), new ComparableVersion("1.0-a1"));
    assertEquals(new ComparableVersion("1.0Alpha1"), new ComparableVersion("1.0-a1"));
    assertEquals(new ComparableVersion("1.0AlphA1"), new ComparableVersion("1.0-a1"));
    assertEquals(new ComparableVersion("1.0BETA1"), new ComparableVersion("1.0-b1"));
    assertEquals(new ComparableVersion("1.0MILESTONE1"), new ComparableVersion("1.0-m1"));
    assertEquals(new ComparableVersion("1.0RC1"), new ComparableVersion("1.0-cr1"));
    assertEquals(new ComparableVersion("1.0GA"), new ComparableVersion("1.0"));
    assertEquals(new ComparableVersion("1.0FINAL"), new ComparableVersion("1.0"));
    assertEquals(new ComparableVersion("1.0-SNAPSHOT"), new ComparableVersion("1-snapshot"));
} 
```

如果版本字符串不能被解析为 major.minor.patch.build，并且无法识别限定符，则整个字符串都被视为限定符。然后将这些限定符作为不区分大小写的字符串进行比较:

```
@Test
public void testQualifierOnly() {
    assertTrue(new ComparableVersion("SomeRandomVersionOne").compareTo(
            new ComparableVersion("SOMERANDOMVERSIONTWO")) < 0);
    assertTrue(new ComparableVersion("SomeRandomVersionThree").compareTo(
            new ComparableVersion("SOMERANDOMVERSIONTWO")) < 0);
} 
```

## 分离器

从数字转换为限定符时，可以选择使用破折号或句点等分隔符:

```
@Test
public void testSeparators() {
    assertEquals(new ComparableVersion("1.0alpha1"), new ComparableVersion("1.0-a1"));
    assertEquals(new ComparableVersion("1.0alpha-1"), new ComparableVersion("1.0-a1"));
    assertEquals(new ComparableVersion("1.0beta1"), new ComparableVersion("1.0-b1"));
    assertEquals(new ComparableVersion("1.0beta-1"), new ComparableVersion("1.0-b1"));
    assertEquals(new ComparableVersion("1.0milestone1"), new ComparableVersion("1.0-m1"));
    assertEquals(new ComparableVersion("1.0milestone-1"), new ComparableVersion("1.0-m1"));
    assertEquals(new ComparableVersion("1.0rc1"), new ComparableVersion("1.0-cr1"));
    assertEquals(new ComparableVersion("1.0rc-1"), new ComparableVersion("1.0-cr1"));
    assertEquals(new ComparableVersion("1.0ga"), new ComparableVersion("1.0"));
} 
```

但是，当从限定符转换为数字时，情况就不一样了:

```
@Test
public void testUnequalSeparators() {
    assertNotEquals(new ComparableVersion("1.0alpha.1"), new ComparableVersion("1.0-a1"));
} 
```

破折号或句点可用于分隔数字:

```
@Test
public void testDashAndPeriod() {
    assertEquals(new ComparableVersion("1-0.ga"), new ComparableVersion("1.0"));
    assertEquals(new ComparableVersion("1.0-final"), new ComparableVersion("1.0"));
    assertEquals(new ComparableVersion("1-0-ga"), new ComparableVersion("1.0"));
    assertEquals(new ComparableVersion("1-0-final"), new ComparableVersion("1-0"));
    assertEquals(new ComparableVersion("1-0"), new ComparableVersion("1.0"));
} 
```

## 长版本

虽然`VersionInformation`类只识别 major.minor.patch.build 格式，但是`ComparableVersion`类可以识别任意数量的数字:

```
@Test
public void testLongVersions() {
    assertEquals(new ComparableVersion("1.0.0.0.0.0.0"), new ComparableVersion("1"));
    assertEquals(new ComparableVersion("1.0.0.0.0.0.0x"), new ComparableVersion("1x"));
} 
```

## 完整的测试

这是用于生成上述示例的完整测试类:

```
package org.apache.maven.artifact.versioning;

import org.junit.Test;

import java.util.Arrays;

import static org.junit.Assert.assertTrue;
import static org.junit.Assert.assertEquals;
import static org.junit.Assert.assertNotEquals;
import static org.junit.Assert.assertArrayEquals;

public class VersionTest {

    private static final ComparableVersion[] VERSIONS = new ComparableVersion[]{
            new ComparableVersion("NotAVersionSting"),
            new ComparableVersion("1.0-alpha"),
            new ComparableVersion("1.0a1-SNAPSHOT"),
            new ComparableVersion("1.0-alpha1"),
            new ComparableVersion("1.0beta1-SNAPSHOT"),
            new ComparableVersion("1.0-b2"),
            new ComparableVersion("1.0-beta3.SNAPSHOT"),
            new ComparableVersion("1.0-beta3"),
            new ComparableVersion("1.0-milestone1-SNAPSHOT"),
            new ComparableVersion("1.0-m2"),
            new ComparableVersion("1.0-rc1-SNAPSHOT"),
            new ComparableVersion("1.0-cr1"),
            new ComparableVersion("1.0-SNAPSHOT"),
            new ComparableVersion("1.0"),
            new ComparableVersion("1.0-sp"),
            new ComparableVersion("1.0-a"),
            new ComparableVersion("1.0-RELEASE"),
            new ComparableVersion("1.0-whatever"),
            new ComparableVersion("1.0.z"),
            new ComparableVersion("1.0.1"),
            new ComparableVersion("1.0.1.0.0.0.0.0.0.0.0.0.0.0.1")

    };

    @Test
    public void ensureArrayInOrder() {
        ComparableVersion[] sortedArray = VERSIONS.clone();
        Arrays.sort(sortedArray);
        assertArrayEquals(VERSIONS, sortedArray);
    }

    @Test
    public void testAliases() {
        assertEquals(new ComparableVersion("1.0-alpha1"), new ComparableVersion("1.0-a1"));
        assertEquals(new ComparableVersion("1.0-beta1"), new ComparableVersion("1.0-b1"));
        assertEquals(new ComparableVersion("1.0-milestone1"), new ComparableVersion("1.0-m1"));
        assertEquals(new ComparableVersion("1.0-rc1"), new ComparableVersion("1.0-cr1"));
    }

    @Test
    public void testDifferentFinalReleases() {
        assertEquals(new ComparableVersion("1.0-ga"), new ComparableVersion("1.0"));
        assertEquals(new ComparableVersion("1.0-final"), new ComparableVersion("1.0"));
    }

    @Test
    public void testQualifierOnly() {
        assertTrue(new ComparableVersion("SomeRandomVersionOne").compareTo(
                new ComparableVersion("SOMERANDOMVERSIONTWO")) < 0);
        assertTrue(new ComparableVersion("SomeRandomVersionThree").compareTo(
                new ComparableVersion("SOMERANDOMVERSIONTWO")) < 0);
    }

    @Test
    public void testSeparators() {
        assertEquals(new ComparableVersion("1.0alpha1"), new ComparableVersion("1.0-a1"));
        assertEquals(new ComparableVersion("1.0alpha-1"), new ComparableVersion("1.0-a1"));
        assertEquals(new ComparableVersion("1.0beta1"), new ComparableVersion("1.0-b1"));
        assertEquals(new ComparableVersion("1.0beta-1"), new ComparableVersion("1.0-b1"));
        assertEquals(new ComparableVersion("1.0milestone1"), new ComparableVersion("1.0-m1"));
        assertEquals(new ComparableVersion("1.0milestone-1"), new ComparableVersion("1.0-m1"));
        assertEquals(new ComparableVersion("1.0rc1"), new ComparableVersion("1.0-cr1"));
        assertEquals(new ComparableVersion("1.0rc-1"), new ComparableVersion("1.0-cr1"));
        assertEquals(new ComparableVersion("1.0ga"), new ComparableVersion("1.0"));
    }

    @Test
    public void testUnequalSeparators() {
        assertNotEquals(new ComparableVersion("1.0alpha.1"), new ComparableVersion("1.0-a1"));
    }

    @Test
    public void testCase() {
        assertEquals(new ComparableVersion("1.0ALPHA1"), new ComparableVersion("1.0-a1"));
        assertEquals(new ComparableVersion("1.0Alpha1"), new ComparableVersion("1.0-a1"));
        assertEquals(new ComparableVersion("1.0AlphA1"), new ComparableVersion("1.0-a1"));
        assertEquals(new ComparableVersion("1.0BETA1"), new ComparableVersion("1.0-b1"));
        assertEquals(new ComparableVersion("1.0MILESTONE1"), new ComparableVersion("1.0-m1"));
        assertEquals(new ComparableVersion("1.0RC1"), new ComparableVersion("1.0-cr1"));
        assertEquals(new ComparableVersion("1.0GA"), new ComparableVersion("1.0"));
        assertEquals(new ComparableVersion("1.0FINAL"), new ComparableVersion("1.0"));
        assertEquals(new ComparableVersion("1.0-SNAPSHOT"), new ComparableVersion("1-snapshot"));
    }

    @Test
    public void testLongVersions() {
        assertEquals(new ComparableVersion("1.0.0.0.0.0.0"), new ComparableVersion("1"));
        assertEquals(new ComparableVersion("1.0.0.0.0.0.0x"), new ComparableVersion("1x"));
    }

    @Test
    public void testDashAndPeriod() {
        assertEquals(new ComparableVersion("1-0.ga"), new ComparableVersion("1.0"));
        assertEquals(new ComparableVersion("1.0-final"), new ComparableVersion("1.0"));
        assertEquals(new ComparableVersion("1-0-ga"), new ComparableVersion("1.0"));
        assertEquals(new ComparableVersion("1-0-final"), new ComparableVersion("1-0"));
        assertEquals(new ComparableVersion("1-0"), new ComparableVersion("1.0"));
    }
} 
```

如果您对 Java 应用程序的自动化部署感兴趣，请注册一个 [Octopus Deploy](https://octopus.com/free) 的入门许可证，并查看一下[我们的文档](https://octopus.com/docs/deployments/java/deploying-java-applications)。