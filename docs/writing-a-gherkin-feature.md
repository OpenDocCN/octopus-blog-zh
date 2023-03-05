# Selenium 系列:编写小黄瓜特性——Octopus Deploy

> 原文：<https://octopus.com/blog/selenium/27-writing-a-gherkin-feature/writing-a-gherkin-feature>

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。

现在我们知道了如何构造正则表达式来将方法映射到小黄瓜步骤，我们可以在`AutomatedBrowserBase`类中为所有适当的方法添加注释。

注意，我们没有为所有的方法添加注释。像`getTextFromElementWithId()`这样返回值的方法不能用在 Gherkin 步骤中，因为 Gherkin 没有变量的概念，所以返回值没有任何意义。我们也不公开像`init()`和`destroy()`这样的方法，因为这些生命周期方法由`openBrowser()`和`closeBrowser()`方法调用。有一些内部方法，像`getWebDriver()`、`getAutomatedBrowser()`、`setAutomatedBrowser()`和`getDesiredCapabilities()`只被装饰者使用，作为小黄瓜步骤公开没有任何意义。

剩下的步骤应用了 Cucumber 注释，分配正则表达式遵循我们在上一篇文章中看到的相同逻辑:

```
package com.octopus.decoratorbase;

import com.octopus.AutomatedBrowser;
import com.octopus.AutomatedBrowserFactory;
import cucumber.api.java.en.And;
import cucumber.api.java.en.Given;
import cucumber.api.java.en.Then;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.DesiredCapabilities;

public class AutomatedBrowserBase implements AutomatedBrowser {

  // ...

  public AutomatedBrowserBase() {

  }

  public AutomatedBrowserBase(AutomatedBrowser automatedBrowser) {
    // ...
  }

  public AutomatedBrowser getAutomatedBrowser() {
    // ...
  }

  public void setAutomatedBrowser(AutomatedBrowser automatedBrowser) {
    // ...
  }

  @Given("^I open the browser \"([^\"]*)\"$")
  public void openBrowser(String browser) {
    // ...
  }

  @Given("^I close the browser$")
  public void closeBrowser() {
    // ...
  }

  @Override
  public WebDriver getWebDriver() {
    // ...
  }

  @Override
  public void setWebDriver(WebDriver webDriver) {
    // ...
  }

  @Override

  public DesiredCapabilities getDesiredCapabilities() {
    // ...
  }

  @Override
  public void init() {
    // ...
  }

  @Override
  public void destroy() {
    // ...
  }

  @And("^I open the URL \"([^\"]*)\"$")
  @Override
  public void goTo(String url) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\"$")
  @Override
  public void clickElementWithId(String id) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void clickElementWithId(String id, int waitTime) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\"$")
  @Override
  public void selectOptionByTextFromSelectWithId(String optionText, String id) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void selectOptionByTextFromSelectWithId(String optionText, String id, int waitTime) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\" with the text \"([^\"]*)\"$")
  @Override
  public void populateElementWithId(String id, String text) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\" with the text \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void populateElementWithId(String id, String text, int waitTime)
  {
    // ...
  }

  @Override
  public String getTextFromElementWithId(String id) {
    // ...
  }

  @Override
  public String getTextFromElementWithId(String id, int waitTime) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the xpath \"([^\"]*)\"$")
  @Override
  public void clickElementWithXPath(String xpath) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the xpath \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void clickElementWithXPath(String xpath, int waitTime) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the xpath \"([^\"]*)\"$")
  @Override
  public void selectOptionByTextFromSelectWithXPath(String optionText, String xpath) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the xpath \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void selectOptionByTextFromSelectWithXPath(String optionText, String xpath, int waitTime) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the xpath \"([^\"]*)\" with the text \"([^\"]*)\"$")
  @Override
  public void populateElementWithXPath(String xpath, String text) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the xpath \"([^\"]*)\" with the text \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void populateElementWithXPath(String xpath, String text, int waitTime) {
    // ...
  }

  @Override
  public String getTextFromElementWithXPath(String xpath) {
    // ...
  }

  @Override
  public String getTextFromElementWithXPath(String xpath, int waitTime) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the css selector \"([^\"]*)\"$")
  @Override
  public void clickElementWithCSSSelector(String cssSelector) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the css selector \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void clickElementWithCSSSelector(String cssSelector, int waitTime) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the css selector \"([^\"]*)\"$")
  @Override
  public void selectOptionByTextFromSelectWithCSSSelector(String optionText, String cssSelector) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the css selector \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void selectOptionByTextFromSelectWithCSSSelector(String optionText, String cssSelector, int waitTime) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the css selector
  \"([^\"]*)\" with the text \"([^\"]*)\"$")
  @Override
  public void populateElementWithCSSSelector(String cssSelector, String text) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the css selector \"([^\"]*)\" with the text \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void populateElementWithCSSSelector(String cssSelector, String text, int waitTime) {
    // ...
  }

  @Override
  public String getTextFromElementWithCSSSelector(String cssSelector) {
    // ...
  }

  @Override
  public String getTextFromElementWithCSSSelector(String cssSelector, int waitTime) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the name \"([^\"]*)\"$")
  @Override
  public void clickElementWithName(String name) {
    // ...
  }

  @And("^I click the \\w+(?:\\s+\\w+)* with the name \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void clickElementWithName(String name, int waitTime) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the name \"([^\"]*)\"$")
  @Override
  public void selectOptionByTextFromSelectWithName(String optionText, String name) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the name \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void selectOptionByTextFromSelectWithName(String optionText, String name, int waitTime) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the name \"([^\"]*)\" with the text \"([^\"]*)\"$")
  @Override
  public void populateElementWithName(String name, String text) {
    // ...
  }

  @And("^I populate the \\w+(?:\\s+\\w+)* with the name \"([^\"]*)\" with the text \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void populateElementWithName(String name, String text, int waitTime) {
    // ...
  }

  @Override
  public String getTextFromElementWithName(String name) {
    // ...
  }

  @Override
  public String getTextFromElementWithName(String name, int waitTime) {
    // ...
  }

  @And("^I click the \"([^\"]*)\" \\w+(?:\\s+\\w+)*$")
  @Override
  public void clickElement(String locator) {
    // ...
  }

  @And("^I click the \"([^\"]*)\" \\w+(?:\\s+\\w+)* waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void clickElement(String locator, int waitTime) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \"([^\"]*)\" \\w+(?:\\s+\\w+)*$")
  @Override
  public void selectOptionByTextFromSelect(String optionText, String locator) {
    // ...
  }

  @And("^I select the option \"([^\"]*)\" from the \"([^\"]*)\" \\w+(?:\\s+\\w+)* waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void selectOptionByTextFromSelect(String optionText, String locator, int waitTime) {
    // ...
  }

  @And("^I populate the \"([^\"]*)\" \\w+(?:\\s+\\w+)* with the text \"([^\"]*)\"$")
  @Override
  public void populateElement(String locator, String text) {
    // ...
  }

  @And("^I populate the \"([^\"]*)\" \\w+(?:\\s+\\w+)* with the text \"([^\"]*)\" waiting up to \"(\\d+)\" seconds?$")
  @Override
  public void populateElement(String locator, String text, int waitTime) {
    // ...
  }

  @Override
  public String getTextFromElement(String locator) {
    // ...
  }

  @Override
  public String getTextFromElement(String locator, int waitTime) {
    // ...
  }

  @And("^I capture the HAR file$")
  @Override
  public void captureHarFile() {
    // ...
  }

  @And("^I capture the complete HAR file$")
  @Override
  public void captureCompleteHarFile() {
    // ...
  }

  @And("^I save the HAR file to \"([^\"]*)\"$")
  @Override
  public void saveHarFile(String file) {
    // ...
  }

  @And("^I block the request to \"([^\"]*)\" returning the HTTP code \"\\d+\"$")
  @Override
  public void blockRequestTo(final String url, final int responseCode) {
    // ...
  }

  @And("^I alter the response fron \"([^\"]*)\" returning the
  HTTP code \"\\d+\" and the response body:$")
  @Override
  public void alterResponseFrom(String url, int responseCode, String responseBody) {
    // ...
  }

  @And("^I maximize the window$")
  @Override
  public void maximizeWindow() {
    // ...
  }

} 
```

有了这些注释，我们现在可以编写一个特性文件来完成从 TicketMonster 购买门票的测试。

将以下代码保存到文件`src/test/resources/com/octopus/ticketmonster.feature`:

```
Feature: Test TicketMonster
  Scenario: Purchase Tickets
    Given I open the browser "ChromeNoImplicitWait"
    When I open the URL "https://ticket-monster.herokuapp.com"
    And I click the "Buy tickets now" button waiting up to "30" seconds
    And I click the "Concert" link waiting up to "30" seconds
    And I click the "Rock concert of the decade" link waiting up to "30" seconds
    And I select the option "Toronto : Roy Thomson Hall" from the "venueSelector" drop-down list waiting up to "30" seconds
    And I click the "bookButton" button waiting up to "30" seconds
    And I select the option "A - Premier platinum reserve" from the "sectionSelect" drop-down list waiting up to "30" seconds
    And I populate the "tickets-1" text box with the text "2" waiting up to "30" seconds
    And I click the "add" button waiting up to "30" seconds
    And I populate the "email" text box with the text "email@example.org" waiting up to "30" seconds
    And I click the "submit" button waiting up to "30" seconds
    Then I close the browser 
```

现在要么从 IntelliJ 运行`CucumberTest`测试类，要么将代码提交给 GitHub，让 Travis CI 为您运行测试。我们刚刚通过 TicketMonster 应用程序成功地复制了这个旅程，这个应用程序是我们在以前的帖子中用 Java 编写的。

如果你大声读出这个测试，它听起来就像是你给同事的指示，如果你在指示他们完成购票的话。但是格式还是有点笨重。大多数步骤以短语`waiting up to "30" seconds`结束，一些定位器如`tickets-1`没有给出很多上下文。

让我们解决短语`waiting up to "30" seconds`的不必要的重复。

我们首先向`AutomatedBrowser`接口添加一个名为`setDefaultExplicitWaitTime()`的新方法。我们将使用这个方法设置一个默认时间，用于所有步骤的显式等待:

```
package com.octopus;

import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.DesiredCapabilities;

public interface AutomatedBrowser {
  // ...
  void setDefaultExplicitWaitTime(int waitTime);
  // ...
} 
```

然后这个方法在`AutomatedBrowserBase`类中实现，并作为一个小黄瓜步骤公开:

```
package com.octopus.decoratorbase;

import com.octopus.AutomatedBrowser;
import com.octopus.AutomatedBrowserFactory;
import cucumber.api.java.en.And;
import cucumber.api.java.en.Given;
import cucumber.api.java.en.Then;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.remote.DesiredCapabilities;

public class AutomatedBrowserBase implements AutomatedBrowser {

  // ...

  @And("^I set the default explicit wait time to \"(\\d+)\" seconds?$")
  @Override
  public void setDefaultExplicitWaitTime(int waitTime) {
    if (getAutomatedBrowser() != null) {
      getAutomatedBrowser().setDefaultExplicitWaitTime(waitTime);
    }
  }

  // ...

} 
```

然后在`WebDriverDecorator`类中，我们捕获了`setDefaultExplicitWaitTime()`方法中的默认等待时间，如果默认等待时间大于 0，则对于之前不接受`waitTime`参数的任何方法使用默认等待时间:

```
package com.octopus.decorators;

import com.octopus.AutomatedBrowser;
import com.octopus.decoratorbase.AutomatedBrowserBase;
import com.octopus.utils.SimpleBy;
import com.octopus.utils.impl.SimpleByImpl;
import org.openqa.selenium.By;
import org.openqa.selenium.WebDriver;
import org.openqa.selenium.support.ui.ExpectedConditions;
import org.openqa.selenium.support.ui.Select;
import org.openqa.selenium.support.ui.WebDriverWait;

public class WebDriverDecorator extends AutomatedBrowserBase {

  // ...

  private int defaultExplicitWaitTime;

  // ...

  @Override
  public void setDefaultExplicitWaitTime(final int waitTime) {
    defaultExplicitWaitTime = waitTime;
  }

  // ...

  @Override
  public void clickElementWithId(final String id) {
    if (defaultExplicitWaitTime <= 0) {
        webDriver.findElement(By.id(id)).click();
      } else {
        clickElementWithId(id, defaultExplicitWaitTime);
    }
  }

  // ...

  @Override
  public void clickElement(final String locator) {
    clickElement(locator, defaultExplicitWaitTime);
  }

  // ...

} 
```

在`setDefaultExplicitWaitTime()`方法中，`defaultExplicitWaitTime`实例变量被设置为默认等待时间:

```
private int defaultExplicitWaitTime;

@Override
public void setDefaultExplicitWaitTime(final int waitTime) {
  defaultExplicitWaitTime = waitTime;
} 
```

然后，我们在任何与元素交互但不接受等待时间参数的方法中使用这个默认值。例如，下面的`clickElementWithId()`方法不接受等待时间参数，默认情况下，会尝试立即单击元素而不等待。

随着我们所做的改变，如果`defaultExplicitWaitTime`大于零，我们改为调用重载的`clickElementWithId()`方法，该方法接受等待时间参数，传入`defaultExplicitWaitTime`的值。这意味着，如果已经定义了`defaultExplicitWaitTime`，那么不接受等待时间参数的方法现在会遵从那些接受等待时间参数的方法的重载版本，并且将依次等待一段时间，以使正在交互的元素可用并处于正确的状态。

所有不接受等待时间参数的方法都用这个新的`if`语句重写了:

```
@Override
public void clickElementWithId(final String id) {
  if (defaultExplicitWaitTime <= 0) {
    webDriver.findElement(By.id(id)).click();
  } else {
    clickElementWithId(id, defaultExplicitWaitTime);
  }
} 
```

唯一不使用相同逻辑来检查`defaultExplicitWaitTime`是否大于 0 的方法是那些使用简化定位器字符串的方法。这些方法已经委托给它们的重载兄弟，等待时间为零，现在被替换为`defaultExplicitWaitTime`变量:

```
@Override
public void clickElement(final String locator) {
  clickElement(locator, defaultExplicitWaitTime);
} 
```

现在我们可以像这样编写小黄瓜特性。我们称该步骤为`And I set the default explicit wait time to "30" seconds`，并从所有其他步骤中删除短语`waiting up to "30" seconds`:

```
Feature: Test TicketMonster With Default Wait
  Scenario: Purchase Tickets with default wait time
    Given I open the browser "ChromeNoImplicitWait"
    And I set the default explicit wait time to "30" seconds
    When I open the URL "https://ticket-monster.herokuapp.com"
    And I click the "Buy tickets now" button
    And I click the "Concert" link
    And I click the "Rock concert of the decade" link
    And I select the option "Toronto : Roy Thomson Hall" from the "venueSelector" drop-down list
    And I click the "bookButton" button
    And I select the option "A - Premier platinum reserve" from the "sectionSelect" drop-down list
    And I populate the "tickets-1" text box with the text "2"
    And I click the "add" button
    And I populate the "email" text box with the text "email@example.org"
    And I click the "submit" button
    Then I close the browser 
```

我们现在非常接近有一个测试，可以用接近简单的英语来写和读。最后剩下的关卡就是像`tickets-1`这样的定位器，很难读懂。

Gherkin 没有任何固有的常量概念，这意味着我们需要引入一些我们称之为别名的东西。别名只不过是键值对，但是它们允许我们将像`Adult Ticket Count`这样有意义的键赋给值`tickets-1`。然后，我们可以在小黄瓜步骤中使用键`Adult Ticket Count`,使该步骤更具可读性。

为了存储这些键值对，我们创建了一个名为`aliases`的新实例变量和一个名为`setAliases()`的新方法来保存它们:

```
public class AutomatedBrowserBase implements AutomatedBrowser {
  // ...

  private Map<String, String> aliases = new HashMap<>();

  // ...

  @Given("^I set the following aliases:$")
  public void setAliases(Map<String, String> aliases) {
    this.aliases.putAll(aliases);
  }

  // ...

} 
```

然后，我们利用 Cucumber 中一个名为数据表的特性来填充别名映射。

注意，正则表达式`^I set the following aliases:$`没有捕获组。传统上，我们使用捕获组作为向方法参数传递值的一种方式。但是在这种情况下，数据表是在步骤之后提供的，并作为一个`Map`对象传递给方法:

```
@Given("^I set the following aliases:$")
public void setAliases(Map<String, String> aliases) {
  this.aliases.putAll(aliases);
} 
```

我们这样称呼这一步。步骤下面的表作为`Map`传递给方法，第一列是键，第二列是值:

```
And I set the following aliases:
  | Venue | venueSelector |
  | Book | bookButton |
  | Section | sectionSelect |
  | Adult Ticket Count | tickets-1 |
  | Add Tickets | add |
  | Checkout | submit | 
```

现在我们可以填充这个地图，我们需要一种方法来阅读它。

Java 8 有一个方便的方法叫做`getOrDefault()`，它允许我们从 map 中获取一个值，或者返回一个默认值。现在，在`AutomatedBrowserBase`类的每个方法中，我们使用字符串值作为别名映射的键，将字符串参数传递给子`AuotomatedBrowser`实例，或者如果别名映射不包含作为键的字符串，则按原样传递参数。

例如，不要调用:

```
getAutomatedBrowser().selectOptionByTextFromSelectWithId(optionText, id) 
```

将参数`optionText`和`id`直接传递给子`AuotomatedBrowser`实例，我们改为调用:

```
getAutomatedBrowser().selectOptionByTextFromSelectWithId(
  aliases.getOrDefault(optionText, optionText),
  aliases.getOrDefault(id, id)) 
```

代码`aliases.getOrDefault(optionText, optionText)`表示“从`aliases`映射中获取分配给键`optionText`的值，或者如果该键不存在，则返回`optionText`作为默认值。”

下面的代码显示了`AutomatedBrowserBase`类中的方法在第一次尝试在别名映射中查找别名值时的样子。每个方法都被更新以查找别名映射，下面的代码显示了`selectOptionByTextFromSelectWithId()`方法是如何更新的:

```
@And("^I select the option \"([^\"]*)\" from the \\w+(?:\\s+\\w+)* with the id \"([^\"]*)\"$")
@Override
public void selectOptionByTextFromSelectWithId(String optionText, String
id) {
  if (getAutomatedBrowser() != null) {
    getAutomatedBrowser().selectOptionByTextFromSelectWithId(
    aliases.getOrDefault(optionText, optionText),
    aliases.getOrDefault(id, id));
  }
} 
```

这些变化意味着我们现在可以像这样编写测试。别名地图现在给模糊的定位器如`tickets-1`一个可读的名字如`Adult Ticket Count`:

```
Feature: Test TicketMonster With Aliases
  Scenario: Purchase Tickets with default wait time and aliases
    Given I open the browser "ChromeNoImplicitWait"
    And I set the following aliases:
      | Venue | venueSelector |
      | Book | bookButton |
      | Section | sectionSelect |
      | Adult Ticket Count | tickets-1 |
      | Add Tickets | add |
      | Checkout | submit |
    And I set the default explicit wait time to "30" seconds
    When I open the URL "https://ticket-monster.herokuapp.com"
    And I click the "Buy tickets now" button
    And I click the "Concert" link
    And I click the "Rock concert of the decade" link
    And I select the option "Toronto : Roy Thomson Hall" from the "Venue" drop-down list
    And I click the "Book" button
    And I select the option "A - Premier platinum reserve" from the "Section" drop-down list
    And I populate the "Adult Ticket Count" text box with the text "2"
    And I click the "Add Tickets" button
    And I populate the "email" text box with the text "email@example.org"
    And I click the "Checkout" button
    Then I close the browser 
```

现在我们有了暴露元素 id 的别名和友好名称的名称，如`Add Tickets`和`Checkout`，测试满足了提供执行测试所需的实现细节的要求，同时也易于阅读。任何熟悉 TicketMonster web 应用程序的人都可以按照这些指示购买音乐会的门票。这就是小黄瓜语言的美妙之处，也是黄瓜库的强大之处。

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。