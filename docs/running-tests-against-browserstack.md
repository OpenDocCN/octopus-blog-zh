# Selenium 系列:对 BrowserStack - Octopus Deploy 运行测试

> 原文：<https://octopus.com/blog/selenium/17-running-tests-against-browserstack/running-tests-against-browserstack>

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。

到目前为止，我们的测试仅限于 Chrome 和 Firefox 等桌面浏览器。根据您运行的操作系统，您也可以测试 Safari、Internet Explorer 和 Edge 等浏览器。但是不管你运行的是哪种操作系统，都没有简单的方法来测试所有流行的浏览器。Windows 不支持 Safari，MacOS 不支持 Internet Explorer 和 Edge，Linux 也不支持这些浏览器。虽然可以在桌面或服务器环境中模拟移动浏览器，但这样做很难配置和维护。

为了解决这些问题，BrowserStack 之类的服务提供了针对大量浏览器运行 WebDriver 测试的能力，包括桌面和移动浏览器。通过管理各种操作系统、浏览器和移动设备，BrowserStack 这样的服务使得大规模跨浏览器测试变得非常容易。

BrowserStack 不是免费服务，要利用它的大部分功能，你需要为一个帐户付费。幸运的是，Mozilla 和微软都与 BrowserStack 合作，提供针对 [Edge](https://www.browserstack.com/test-on-microsoft-edge-browser) 和 [Firefox](https://blog.mozilla.org/blog/2017/03/03/mozilla-browserstack-partner-drive-mobile-testing-real-devices/) 浏览器的免费测试。我们将利用这个服务来构建一些可以免费运行的远程测试。好消息是，一旦你在 Edge 或 Firefox 这样的浏览器上运行了测试，重用这些代码来运行 BrowserStack 提供的任何其他浏览器的测试都是非常简单的。

要创建 BrowserStack 帐户，请前往[https://www.browserstack.com](https://www.browserstack.com)。在主页或顶部菜单中，您会看到注册免费试用的链接。

输入您的电子邮件地址、密码和姓名，并继续下一页。

[![](img/69ee61877e2e0f67707984c38c73d14d.png)](#)

仅此而已。您现在有一个 BrowserStack 帐户。

[![](img/4ccbfd5e54853e70393ca0debde37c72.png)](#)

为了连接到 BrowserStack，我们需要获取访问密钥。这可以通过点击`Account`菜单并选择`Settings`找到。

[![](img/b400436e434ec0cda60109110825bf83.png)](#)

您将在 Automate 标题下找到访问键。记下`Username`和`Access Key`，因为我们稍后将需要这些值。

[![](img/377a041b8ee59184a8759c90c51d58c9.png)](#)

为了对 BrowserStack 远程运行测试，我们需要创建一个`RemoteDriver`类的实例。与`ChromeDriver`或`FirefoxDiver`不同的是，`RemoteDriver`被设计用来控制远程服务器上的浏览器。这意味着我们需要给`RemoteDriver`一个 URL 来发送命令和凭证。

https://www.browserstack.com/automate/java 的 BrowserStack 文档显示了我们需要连接的 URL。它的格式是:`https://<username>:<password>@hub-cloud.browserstack.com/wd/hub`。用户名和密码嵌入在 URL 中。

为了使测试能够在 BrowserStack 上运行，我们将创建一个名为`BrowserStackDecorator`的新装饰器:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import com.octopus.exceptions.ConfigurationException;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.RemoteWebDriver;
import java.net.MalformedURLException;

import java.net.URL;

public class BrowserStackDecorator extends AutomatedBrowserBase {

  private static final String USERNAME_ENV = "BROWSERSTACK_USERNAME";
  private static final String AUTOMATE_KEY_ENV = "BROWSERSTACK_KEY";

  public BrowserStackDecorator(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  }

  @Override
  public void init() {
    try {
      final String url = "https://" +
        System.getenv(USERNAME_ENV) + ":" +
        System.getenv(AUTOMATE_KEY_ENV) +
        "@hub-cloud.browserstack.com/wd/hub";
      final WebDriver webDriver = new RemoteWebDriver(new URL(url), getDesiredCapabilities());
      getAutomatedBrowser().setWebDriver(webDriver);
      getAutomatedBrowser().init();
    } catch (MalformedURLException ex) {
      throw new ConfigurationException(ex);
    }
  }
} 
```

因为将密码嵌入到应用程序中被认为是不好的做法，所以我们将从环境变量中获取用户名和密码。用户名将在`BROWSERSTACK_USERNAME`环境变量中找到，而密码将在`BROWSERSTACK_KEY`环境变量中找到。我们将为这些字符串创建常量，以便稍后在代码中访问它们:

```
private static final String USERNAME_ENV = "BROWSERSTACK_USERNAME";
private static final String AUTOMATE_KEY_ENV = "BROWSERSTACK_KEY"; 
```

接下来，我们构建允许`RemoteDriver`联系 BrowserStack 服务的 URL。我们使用对`System.getenv()`的调用从环境变量中获取用户名和密码:

```
final String url = "https://" +
  System.getenv(USERNAME_ENV) + ":" +
  System.getenv(AUTOMATE_KEY_ENV) +
  "@hub-cloud.browserstack.com/wd/hub"; 
```

在`RemoteDriver`类的构造和类似`ChromeDriver`的类之间只有很小的区别。

`RemoteDriver()`构造函数获取要连接的服务的 URL 和所需的功能。

对于`RemoteDriver`类，没有类似于`ChromeOptions`的等价类；它直接使用`DesiredCapabilities`对象:

```
final WebDriver webDriver = new RemoteWebDriver(new URL(url), getDesiredCapabilities()); 
```

如果我们构造的 URL 无效，就会抛出一个`MalformedURLException`。我们捕获这个异常，并将其包装在一个名为`ConfigurationException`的未检查异常中:

```
catch (MalformedURLException ex) {
  throw new ConfigurationException(ex);
} 
```

`ConfigurationException`类用于指示所需的环境变量尚未配置:

```
package com.octopus.exceptions;

public class ConfigurationException extends RuntimeException {
  public ConfigurationException() {

  }

  public ConfigurationException(final String message) {
    super(message);
  }

  public ConfigurationException(final String message, final Throwable ex)
  {
    super(message, ex);
  }

  public ConfigurationException(final Exception ex) {
    super(ex);
  }
} 
```

建造`RemoteDriver`只是故事的一半。因为`RemoteDriver`是远程服务所公开的任何浏览器的通用接口，所以我们将想要测试的浏览器的细节定义为所需的 capabilities 对象中的值。BrowserStack 有一个在线工具，可以在[https://www.browserstack.com/automate/capabilities](https://www.browserstack.com/automate/capabilities)生成这些所需的功能设置。您选择所需的操作系统、设备或浏览器以及一些其他设置，如屏幕分辨率，该工具将生成可用于填充`DesiredCapabilities`对象的代码。

我们将首先针对 Windows 10 中提供的 Edge 浏览器进行测试。

[![](img/7bbd7edddcc07f8a4d427d307a6a7dea.png)](#)

这些期望的功能设置将在一个名为`BrowserStackEdgeDecorator`的新装饰器中定义:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import org.openqa.selenium.remote.DesiredCapabilities;

public class BrowserStackEdgeDecorator extends AutomatedBrowserBase {

    public BrowserStackEdgeDecorator(final AutomatedBrowser  automatedBrowser) {
        super(automatedBrowser);
    }

    @Override
    public DesiredCapabilities getDesiredCapabilities() {
        final DesiredCapabilities caps = getAutomatedBrowser().getDesiredCapabilities();

        caps.setCapability("os", "Windows");
        caps.setCapability("os_version", "10");
        caps.setCapability("browser", "Edge");
        caps.setCapability("browser_version", "insider preview");
        caps.setCapability("browserstack.local", "false");
        caps.setCapability("browserstack.selenium_version", "3.7.0");
        return caps;
    }
} 
```

为了将这两个新的装饰器结合在一起，我们在工厂中创建了一种新型的浏览器。

注意，在构建`AutomatedBrowser`实例来运行 BrowserStack 中的测试时，我们不使用`BrowserMobDecorator`类。BrowserMob 只对运行在本地机器上的浏览器可用，因为它被绑定到 localhost 接口上的一个端口，这意味着它不会像那些运行在 BrowserStack 平台上的浏览器那样暴露给外部浏览器。为了避免给远程浏览器配置他们无权访问的本地代理，我们将`BrowserMobDecorator`类排除在装饰链之外。

在这里，我们嵌套装饰器的顺序很重要。`BrowserStackDecorator`期望在嵌套装饰器中设置期望的功能，在本例中是`BrowserStackEdgeDecorator`。这意味着`BrowserStackDecorator`必须将`BrowserStackEdgeDecorator`传递给它的构造函数，而不是相反:

```
package com.octopus;

import com.octopus.decorators.*;

public class AutomatedBrowserFactory {

  public AutomatedBrowser getAutomatedBrowser(String browser) {

    // ...

    if ("BrowserStackEdge".equalsIgnoreCase(browser)) {
      return getBrowserStackEdge();
    }

    if ("BrowserStackEdgeNoImplicitWait".equalsIgnoreCase(browser)) {
      return getBrowserStackEdgeNoImplicitWait();
    }

    throw new IllegalArgumentException("Unknown browser " + browser);

  }

  // ...

  private AutomatedBrowser getBrowserStackEdge() {
    return new BrowserStackDecorator(
      new BrowserStackEdgeDecorator(
        new ImplicitWaitDecorator(10,
          new WebDriverDecorator()
        )
      )
    );
  }

  private AutomatedBrowser getBrowserStackEdgeNoImplicitWait() {
    return new BrowserStackDecorator(
      new BrowserStackEdgeDecorator(
        new WebDriverDecorator()
      )
    );
  }
} 
```

现在我们可以在测试中使用这个新的浏览器。请注意，我们不是从本地磁盘访问 HTML 文件，而是打开 URL[https://s3 . amazonaws . com/web driver-testing-website/form . HTML](https://s3.amazonaws.com/webdriver-testing-website/form.html)，这是我们在之前的帖子中在 S3 上传的文件的 URL。

如果我们尝试对本地文件运行测试，就会失败。这是因为远程浏览器试图打开的 URL 看起来类似于`file:///Users/username/javaproject/src/test/resources/form.html`(取决于您的本地操作系统)。运行远程浏览器的操作系统上不存在该文件，因为远程浏览器运行在由 BrowserStack 管理的操作系统上。任何加载该文件的尝试都会失败:

```
@Test
public void browserStackTest() {

    final AutomatedBrowser automatedBrowser =
            AUTOMATED_BROWSER_FACTORY.getAutomatedBrowser("BrowserStackEdge");

    final String formButtonLocator = "button_element";
    final String formTextBoxLocator = "text_element";
    final String formTextAreaLocator = "textarea_element";
    final String formDropDownListLocator = "[name=select_element]";
    final String formCheckboxLocator = "//*[@name=\"checkbox1_element\"]";

    final String messageLocator = "message";

    try {
        automatedBrowser.init();

        automatedBrowser.goTo("https://s3.amazonaws.com/webdriver-testing-website/form.html");

        automatedBrowser.clickElement(formButtonLocator);
        assertEquals("Button Clicked", automatedBrowser.getTextFromElement(messageLocator));

        automatedBrowser.populateElement(formTextBoxLocator, "test text");
        assertEquals("Text Input Changed", automatedBrowser.getTextFromElement(messageLocator));

        automatedBrowser.populateElement(formTextAreaLocator, "test text");
        assertEquals("Text Area Changed", automatedBrowser.getTextFromElement(messageLocator));

        automatedBrowser.selectOptionByTextFromSelect("Option 2.1", formDropDownListLocator);
        assertEquals("Select Changed",  automatedBrowser.getTextFromElement(messageLocator));

        automatedBrowser.clickElement(formCheckboxLocator);
        assertEquals("Checkbox Changed",  automatedBrowser.getTextFromElement(messageLocator));

    } finally {
        automatedBrowser.destroy();
    }
} 
```

运行该测试将生成如下异常:

```
org.openqa.selenium.WebDriverException: Invalid username or password
(WARNING: The server did not provide any stacktrace information) Command
duration or timeout: 1.81 seconds Build info: version: '3.11.0',
revision: 'e59cfb3', time: '2018-03-11T20:26:55.152Z' System info:
host: 'Christinas-MBP', ip: '192.168.1.84', os.name: 'Mac OS X',
os.arch: 'x86_64', os.version: '10.13.4', java.version:
'1.8.0_144' Driver info: driver.version: RemoteWebDriver at
sun.reflect.NativeConstructorAccessorImpl.newInstance0(Native Method) at
sun.reflect.NativeConstructorAccessorImpl.newInstance(NativeConstructorAccessorImpl.java:62)
at
sun.reflect.DelegatingConstructorAccessorImpl.newInstance(DelegatingConstructorAccessorImpl.java:45)
at java.lang.reflect.Constructor.newInstance(Constructor.java:423) at
org.openqa.selenium.remote.ErrorHandler.createThrowable(ErrorHandler.java:214)
at
org.openqa.selenium.remote.ErrorHandler.throwIfResponseFailed(ErrorHandler.java:166) 
```

这是因为我们没有用 BrowserStack 用户名和密码设置环境变量。要将这些环境变量添加到测试中，点击 IntelliJ 中包含配置的下拉列表，并点击`Edit Configurations...`

[![](img/6ba39455b88b7e568c29a3dedf59be02.png)](#)

在 Configuration 选项卡下，您会看到一个名为`Environment Variables`的字段。单击该字段右侧的按钮。

[![](img/4b70ddb1fc673e87548ca2a875ff5299.png)](#)

在对话框中输入环境变量，并保存更改。点击`OK`按钮两次保存更改。

[![](img/2a2ed4e416fc26de4d62b0b8edf9c0d7.png)](#)

这一次，测试将成功运行。您可以通过登录 BrowserStack 并点击产品➜自动化链接来查看测试运行情况。默认情况下，显示最后一次测试。右边的屏幕将向您显示针对远程浏览器正在运行的测试，或者如果测试已经完成，它将提供测试的录制视频。

试用 BrowserStack 帐户通常可以获得 100 分钟的免费时间，但由于 BrowserStack 和微软之间的合作关系，这些时间不会被针对 Edge 运行的测试所消耗。

既然我们有能力在 Edge 浏览器上运行测试，那么开始在 BrowserStack 提供的任何其他浏览器上运行测试就非常简单了。我们将在下一篇文章中看到这一点，届时我们将对作为 BrowserStack 服务的一部分提供的移动设备进行测试。

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。