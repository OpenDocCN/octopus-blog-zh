# Selenium 系列:无头浏览器- Octopus Deploy

> 原文：<https://octopus.com/blog/selenium/10-headless-browsers/headless-browsers>

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。

你现在应该已经注意到，用 WebDriver 运行测试会打开一个浏览器窗口，加载网页并与之进行交互，就好像通过某种看不见的鼠标指针一样。虽然在浏览器中观察测试的进展是很有用的，但有时还是希望测试在屏幕外完成。例如，作为连续部署过程的一部分运行测试，不需要任何人在测试执行时监视浏览器。事实上，有时甚至没有一个监视器连接到运行测试的系统上；这就是所谓的无头环境。那么我们如何在这样的无头环境中运行测试呢？

像 [PhantomJS](http://phantomjs.org/) 这样的项目就是为了解决这个问题而创建的。PhantomJS 是一个基于 WebKit 的 web 浏览器，WebKit 是支持 Apple Safari 等浏览器的库。然而与传统浏览器不同的是，PhantomJS 没有 GUI，它被设计为由 WebDriver 等技术控制。因为它没有 GUI，所以 PhantomJS 可以在持续集成服务器上运行，而这些服务器传统上是在无头服务器上托管的。这意味着您可以在中央服务器上运行 WebDriver 测试来响应应用程序的更改，而不必在桌面环境中启动浏览器窗口。

最近，像 Firefox 和 Chrome 这样的浏览器增加了对无头浏览的本地支持。这对任何编写 WebDriver 测试的人来说都是一个很大的好处，因为这意味着测试可以在最终用户已经安装的浏览器上运行，同时仍然允许测试在一个无头服务器上运行。

如今幻想曲的发展已经停滞。项目的一个维护者[已经下台](https://groups.google.com/forum/#!topic/phantomjs/9aI5d-LDuNE)，PhantomJS 最新发布已经 2 年多了。但好消息是，配置 Chrome 和 Firefox 在无头环境中运行测试非常容易。

在我们开始配置无头浏览器之前，我们需要为配置驱动程序类添加一些额外的支持。

WebDriver 使用一个名为`DesiredCapabilities`的类作为浏览器驱动设置的通用容器。`DesiredCapabilities`类本质上是一个键/值对的容器，带有一些配置常用设置的方便方法。

首先，我们将方法`getDesiredCapabilities()`添加到`AutomatedBrowser`接口:

```
public interface AutomatedBrowser {
  // ...
  DesiredCapabilities getDesiredCapabilities();
  // ...
} 
```

然后我们在`AutomatedBrowserBase`类中添加一个默认方法。

该方法与典型的默认装饰方法实现略有不同，如果没有父`AutomatedBrowser`实例返回`DesiredCapabilities`类的实例，我们返回一个新的`DesiredCapabilities`实例，而不是`null`。这确保了如果没有装饰者提供任何`DesiredCapabilities`，我们总是可以依赖返回的默认实例:

```
@Override
public DesiredCapabilities getDesiredCapabilities() {
  if (getAutomatedBrowser() != null) {
    return getAutomatedBrowser().getDesiredCapabilities();
  }

  return new DesiredCapabilities();
} 
```

`DesiredCapabilities`类用于所有浏览器通用的配置设置。每个驱动程序都有一个相应的“选项”类，用于配置浏览器的特定设置。这两个对象合并在一起以构建完整的配置设置集。

下面是为支持这两个配置类而更新的`ChromeDecorator`类的代码。我们创建了一个`ChromeOptions`类的实例，`merge()`用`getDesiredCapabilities()`返回的公共设置，并将合并的结果传递给`ChromeDriver()`构造函数。

这段代码还没有配置任何额外的设置，但是它演示了如何将`DesiredCapabilities`类与浏览器特定的选项类结合使用:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public class ChromeDecorator extends AutomatedBrowserBase {

    public ChromeDecorator(final AutomatedBrowser automatedBrowser) {
        super(automatedBrowser);
    }

    @Override
    public void init() {
        final ChromeOptions options = new ChromeOptions();
        options.merge(getDesiredCapabilities());
        final WebDriver webDriver = new ChromeDriver(options);
        getAutomatedBrowser().setWebDriver(webDriver);
        getAutomatedBrowser().init();
    }
} 
```

我们对`FirefoxDecorator`类采用相同的模式，将`FirefoxDriver`类与`DesiredCapabilities`类合并:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;

public class FirefoxDecorator extends AutomatedBrowserBase {
  public FirefoxDecorator(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  }

  @Override
  public void init() {
    final FirefoxOptions options = new FirefoxOptions();
    options.merge(getDesiredCapabilities());
    final WebDriver webDriver = new FirefoxDriver(options);
    getAutomatedBrowser().setWebDriver(webDriver);
    getAutomatedBrowser().init();
  }
} 
```

通过配置`ChromeOptions`或`FirefoxOptions`实例，可以在无头模式下启动浏览器。

为了在无头模式下启动 Chrome，我们将一些参数传递给`chrome`可执行文件。`ChromeOptions`类通过方法`setHeadless()`提供了配置这些参数的简单方法。

让我们来看看允许我们在无头模式下运行 Chrome 的`ChromeDecorator`类的代码:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.chrome.ChromeDriver;
import org.openqa.selenium.chrome.ChromeOptions;

public class ChromeDecorator extends AutomatedBrowserBase {

  final boolean headless;

  public ChromeDecorator(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
    this.headless = false;
  }

  public ChromeDecorator(final boolean headless, final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
    this.headless = headless;
  }

  @Override
  public void init() {
    final ChromeOptions options = new ChromeOptions();
    options.setHeadless(headless);
    options.merge(getDesiredCapabilities());
    final WebDriver webDriver = new ChromeDriver(options);
    getAutomatedBrowser().setWebDriver(webDriver);
    getAutomatedBrowser().init();

  }
} 
```

首先，我们提供一个名为`headless`的实例变量来跟踪 Chrome 浏览器的这个实例是否应该以无头模式运行。为了设置这个变量，我们重载构造函数:

```
final boolean headless;

public ChromeDecorator(final AutomatedBrowser automatedBrowser) {
  super(automatedBrowser);
  this.headless = false;
}

public ChromeDecorator(final boolean headless, final AutomatedBrowser automatedBrowser) {
  super(automatedBrowser);
  this.headless = headless;
} 
```

在`init()`方法中，我们调用`setHeadless()`来启用或禁用无头模式(尽管默认情况下给定的无头模式是禁用的，调用`setHeadless(false)`不会改变任何事情):

```
options.setHeadless(headless); 
```

看一下`ChomeOptions.setHeadless()`方法，我们可以看到通过将`--headless`和`--disable-gpu`参数传递给 Chrome 来启用无头模式:

```
public ChromeOptions setHeadless(boolean headless) {

  args.remove("--headless");
  if (headless) {
    args.add("--headless");
    args.add("--disable-gpu");
  }

  return this;
} 
```

然后，我们用一个参数来更新`AutomatedBrowserFactory getChromeBrowser()`方法，以定义 Chrome 浏览器是否应该是无头的:

```
private AutomatedBrowser getChromeBrowser(final boolean headless) {
  return new ChromeDecorator(headless,
    new ImplicitWaitDecorator(10,
      new WebDriverDecorator()
    )
  );
} 
```

最后，我们更新了`getAutomatedBrowser()`方法，允许创建一个 Chrome 的无头实例:

```
public AutomatedBrowser getAutomatedBrowser(String browser) {
  if ("Chrome".equalsIgnoreCase(browser)) {
    return getChromeBrowser(false);
  }

  if ("ChromeHeadless".equalsIgnoreCase(browser)) {
    return getChromeBrowser(true);
  }

  // ...
} 
```

有了这些变化，我们就可以更新测试，在 Chrome 的一个无头实例上运行它们。

当测试运行时，您将看不到浏览器窗口。但是测试将在后台执行，并像以前一样通过:

```
@Test
public void formTestByIDHeadless() throws URISyntaxException {
  final AutomatedBrowser automatedBrowser =
    AUTOMATED_BROWSER_FACTORY.getAutomatedBrowser("ChromeHeadless");

  // ...
} 
```

创建 Firefox 的无头实例的过程与 Chrome 几乎完全相同。

首先用一个设置`headless`实例变量的构造函数来更新`FirefoxDecorator`类，并且调用 options 类中的`setHeadless()`来配置驱动程序上的无头模式:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.firefox.FirefoxDriver;
import org.openqa.selenium.firefox.FirefoxOptions;

public class FirefoxDecorator extends AutomatedBrowserBase {

  final boolean headless;

  public FirefoxDecorator(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
    this.headless = false;
  }

  public FirefoxDecorator(final boolean headless, final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
    this.headless = headless;
  }

  @Override
  public void init() {
    final FirefoxOptions options = new FirefoxOptions();
    options.setHeadless(headless);
    options.merge(getDesiredCapabilities());
    final WebDriver webDriver = new FirefoxDriver(options);
    getAutomatedBrowser().setWebDriver(webDriver);
    getAutomatedBrowser().init();
  }
} 
```

查看`FirefoxOptions.setHeadless()`方法，我们可以看到通过将`-headless`参数传递给 Firefox 来启用无头模式:

```
public FirefoxOptions setHeadless(boolean headless) {
  args.remove("-headless");
  if (headless) {
    args.add("-headless");
  }
  return this;
} 
```

然后更新`AutomatedBrowserFactory getFirefoxBrowser()`方法以支持设置无头模式:

```
private AutomatedBrowser getFirefoxBrowser(final boolean headless) {
  return new FirefoxDecorator(headless,
    new ImplicitWaitDecorator(10,
      new WebDriverDecorator()
    )
  );
} 
```

并且更新了`getAutomatedBrowser()`方法以支持创建 Firefox 的 headless 实例:

```
public AutomatedBrowser getAutomatedBrowser(String browser) {
  //...
  if ("Firefox".equalsIgnoreCase(browser)) {
    return getFirefoxBrowser(false);
  }

  if ("FirefoxHeadless".equalsIgnoreCase(browser)) {
    return getFirefoxBrowser(true);
  }

  //...
} 
```

然后，就像 Chrome 浏览器一样，测试可以更新为使用 Firefox 的 headless 版本:

```
@Test
public void formTestByIDHeadlessFirefox() throws URISyntaxException {
  final AutomatedBrowser automatedBrowser =
    AUTOMATED_BROWSER_FACTORY.getAutomatedBrowser("FirefoxHeadless");

  // ...
} 
```

在像 PhantomJS 这样行为不太像真正的浏览器的专用浏览器上运行测试曾经是测试人员的一个痛点，但却是一个不可避免的祸害。通过支持无头浏览，Chrome 和 Firefox 等浏览器为测试人员在无头服务器上的自动化测试中使用最终用户使用的相同浏览器铺平了道路。当我们与 Travis CI 和 AWS Lambda 等平台集成时，我们将在以后的帖子中利用这些无头浏览器。

此外，通过暴露通过`DesiredCapabilities`类配置浏览器的能力，我们提供了一个钩子，我们可以利用新的 decorators 来添加功能，如自定义代理，这正是我们将在下一篇文章中做的。

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。