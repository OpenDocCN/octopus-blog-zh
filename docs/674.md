# Selenium 系列:页面对象模型设计模式- Octopus Deploy

> 原文：<https://octopus.com/blog/selenium/23-the-page-object-model-pattern/the-page-object-model-pattern>

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。

虽然我们之前的测试成功地验证了在 TicketMonster 中购买活动门票的过程，但是这种测试方式(我们按照顺序定义与页面的每个交互)有一些局限性。

第一个限制是每个交互都不是特别描述性的。对被测应用程序了解有限的人会被一行代码弄糊涂，这是可以理解的:

```
automatedBrowser.populateElement("tickets-1", "2", WAIT_TIME); 
```

这种情况下的`tickets-1`是什么？这个标识符没有很好地描述它所定位的元素。您需要非常熟悉被测试的应用程序，才能知道这行代码是做什么的。

第二个，也许是最重要的，限制是用来构建这个测试的代码是不可重用的。想象一下，除了这个购买机票的测试之外，我们想要编写第二个测试来验证每个可用部分的机票价格。您可以编写一个这样的测试来确保价格变化不会导致不合理的高或低票价。

为了编写第二个测试，与应用程序的每个交互都需要复制并粘贴到第二个测试方法中。然而，最好避免复制和粘贴，因为这会使代码更难维护，因为公共功能现在存在于多个方法中，并且都必须单独更新。

解决这两个限制的一个方法是使用页面对象模型(POM)设计模式。POM 设计模式封装了一个类中与单个页面的交互。这意味着与页面的交互现在暴露在具有友好名称的方法后面，并且这些类可以在测试之间重用。

让我们看看如何使用 POM 设计模式重写测试。

我们创建的每个 POM 类都需要访问一个`AutomatedBrowser`实例。此外，我们将为与元素交互时使用的显式等待定义一个公共等待时间。公开这些共享属性是通过一个名为`BasePage`的类来完成的。

请注意，实例变量、静态变量和构造函数都有受保护的范围。这意味着它们只对扩展了`BasePage`的类可用，意味着`BasePage`不是我们可以直接实例化的东西:

```
package com.octopus.pages;

import com.octopus.AutomatedBrowser;

public class BasePage {

  protected static final int WAIT_TIME = 30;

  protected final AutomatedBrowser automatedBrowser;

  protected BasePage(AutomatedBrowser automatedBrowser) {
    this.automatedBrowser = automatedBrowser;
  }
} 
```

我们在 TicketMonster 应用程序的主页上开始所有的测试。为了表示这个主页，我们创建了类`MainPage`:

```
package com.octopus.pages.ticketmonster;

import com.octopus.AutomatedBrowser;

import com.octopus.pages.BasePage;

public class MainPage extends BasePage {

  private static final String URL =
    "https://ticket-monster.herokuapp.com";

  private static final String BUY_TICKETS_NOW = "Buy tickets now";

  public MainPage(final AutomatedBrowser automatedBrowser) {

  super(automatedBrowser);

  }

  public MainPage openPage() {
    automatedBrowser.goTo(URL);
    return this;
  }

  public EventsPage buyTickets() {
    automatedBrowser.clickElement(BUY_TICKETS_NOW, WAIT_TIME);
    return new EventsPage(automatedBrowser);
  }
} 
```

让我们来分解这个代码。

我们所有的 POM 类都将扩展`BasePage`。扩展`BasePage`使这些类可以访问`AutomatedBrowser`的实例，并使用共享的默认等待时间:

```
public class MainPage extends BasePage { 
```

为了使 URL 和元素标识符更易维护，我们将字符串赋给常量。使用常量变量意味着我们可以给这些字符串一些有意义的上下文，这在以后处理一些更不直观的元素定位符时很重要:

```
private static final String URL =
  "https://ticket-monster.herokuapp.com";

private static final String BUY_TICKETS_NOW = "Buy tickets now"; 
```

构造函数获取`AutomatedBrowser`的一个实例，并将其传递给`BasePage`构造函数:

```
public MainPage(final AutomatedBrowser automatedBrowser) {
  super(automatedBrowser);
} 
```

我们定义的第一个方法将打开应用程序主页的 URL。为了完成这个动作，我们创建了一个名为`openPage()`的方法。

在像`openPage()`这样的方法中，要打开的 URL 或要交互的元素的具体细节被封装。这个方法的调用者不需要知道正在打开的 URL，如果 URL 改变了，只需要在一个位置改变，使得维护更加容易。

为了允许链接调用，我们返回一个实例`this`作为最终语句。POM 类中的每个方法都将返回 POM 类，可以在其上执行额外的交互。这样，POM 类的消费者可以很容易地理解应用程序的流程，我们将在编写测试方法时看到这一点:

```
public MainPage openPage() {
  automatedBrowser.goTo(URL);
  return this;
} 
```

我们在主页上唯一感兴趣的动作是点击`Buy tickets now`链接，我们用一个叫做`buyTickets()`的方法来做。

如果您还记得上一篇文章，我们如何与像这个链接这样的元素交互并不像看起来那么简单，因为这些元素可以是样式化的链接(`<a>`元素)，也可以是表单按钮(`<input>`元素)。根据使用的元素类型，我们的第一个测试必须使用不同的定位器。链接可以通过它们的文本来识别，而表单按钮必须通过一个`id`或`name`属性来识别。

元素之间的区别不再是编写这些测试的人所关心的，而是被封装在这个 POM 类中。调用`buyTickets()`方法将导致所需的元素被点击，不管该元素是如何定位的。

因为单击此链接会将浏览器定向到事件页面，所以我们返回了一个`EventsPage`类的实例。`buyTickets()`的调用者可以理解，这个返回值表明页面导航已经发生，现在必须使用`EventsPage`类来执行与页面的进一步交互:

```
public EventsPage buyTickets() {
  automatedBrowser.clickElement(BUY_TICKETS_NOW, WAIT_TIME);
  return new EventsPage(automatedBrowser);
} 
```

让我们来看看`EventsPage`类:

```
package com.octopus.pages.ticketmonster;

import com.octopus.AutomatedBrowser;

import com.octopus.pages.BasePage;

public class EventsPage extends BasePage {

  public EventsPage(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  }

  public VenuePage selectEvent(final String category, final String event)
  {
    automatedBrowser.clickElement(category, WAIT_TIME);
    automatedBrowser.clickElement(event, WAIT_TIME);
    return new VenuePage(automatedBrowser);
  }
} 
```

与`MainPage`类一样，`EventsPage`扩展了`BasePage`类，并将`AutomatedBrowser`的实例从其构造函数传递给`BaseClass`构造函数:

```
public class EventsPage extends BasePage {
  public EventsPage(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  } 
```

在活动页面上，我们唯一想做的事情是选择我们要购买门票的活动。这包括单击左侧可折叠菜单中的链接。

为了允许该方法选择菜单中的任何选项，我们将菜单类别和事件名称作为参数传入。

选择一个事件将触发应用程序加载 venue 页面。我们通过返回一个`VenuePage`类的实例来表示这种导航:

```
public VenuePage selectEvent(final String category, final String event)
{
  automatedBrowser.clickElement(category, WAIT_TIME);
  automatedBrowser.clickElement(event, WAIT_TIME);
  return new VenuePage(automatedBrowser);
} 
```

让我们来看看`VenuePage`类:

```
package com.octopus.pages.ticketmonster;

import com.octopus.AutomatedBrowser;
import com.octopus.pages.BasePage;

public class VenuePage extends BasePage {

  private static final String VENUE_DROP_DOWN_LIST = "venueSelector";
  private static final String BOOK_BUTTON = "bookButton";

  public VenuePage(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  }

  public VenuePage selectVenue(final String venue) {
    automatedBrowser.selectOptionByTextFromSelect(venue, VENUE_DROP_DOWN_LIST, WAIT_TIME);
    return this;
  }

  public CheckoutPage book() {
    automatedBrowser.clickElement(BOOK_BUTTON, WAIT_TIME);
    return new CheckoutPage(automatedBrowser);
  }
} 
```

这个类扩展了`BasePage`，将`AutomatedBrowser`传递给`BasePage`构造函数，并为 venue 下拉列表和 book 按钮的定位器定义了一些常量:

```
public class VenuePage extends BasePage {
  private static final String VENUE_DROP_DOWN_LIST = "venueSelector";
  private static final String BOOK_BUTTON = "bookButton";

  public VenuePage(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  } 
```

通过`selectVenue()`方法选择场地，场地名称作为参数传入。

选择一个地点不会触发任何页面导航，因此我们返回`this`以指示未来的交互仍将在同一页面上执行:

```
public VenuePage selectVenue(final String venue) {
  automatedBrowser.selectOptionByTextFromSelect(venue,
  VENUE_DROP_DOWN_LIST, WAIT_TIME);

  return this;
} 
```

通过`book()`方法进入预订页面。

单击 book 按钮会导致应用程序导航到 checkout 页面，我们通过返回一个`CheckoutPage`类的实例来表示它:

```
public CheckoutPage book() {
  automatedBrowser.clickElement(BOOK_BUTTON, WAIT_TIME);
  return new CheckoutPage(automatedBrowser);
} 
```

让我们来看看`CheckoutPage`类:

```
package com.octopus.pages.ticketmonster;

import com.octopus.AutomatedBrowser;

import com.octopus.pages.BasePage;

public class CheckoutPage extends BasePage {

    private static final String SECTION_DROP_DOWN_LIST = "sectionSelect";
    private static final String ADULT_TICKET_COUNT = "tickets-1";
    private static final String ADD_TICKETS_BUTTON = "add";
    private static final String EMAIL_ADDRESS = "email";
    private static final String CHECKOUT_BUTTON = "submit";

    public CheckoutPage(final AutomatedBrowser automatedBrowser) {
        super(automatedBrowser);
    }

    public CheckoutPage buySectionTickets(final String section, final
    Integer adultCount) {
        automatedBrowser.selectOptionByTextFromSelect(section, SECTION_DROP_DOWN_LIST, WAIT_TIME);
        automatedBrowser.populateElement(ADULT_TICKET_COUNT, adultCount.toString(), WAIT_TIME);
        automatedBrowser.clickElement(ADD_TICKETS_BUTTON, WAIT_TIME);
        return this;
    }

    public ConfirmationPage checkout(final String email) {
        automatedBrowser.populateElement(EMAIL_ADDRESS, email, WAIT_TIME);
        automatedBrowser.clickElement(CHECKOUT_BUTTON, WAIT_TIME);
        return new ConfirmationPage(automatedBrowser);
    }
} 
```

这个类扩展了`BasePage`并将`AutomatedBrowser`传递给`BasePage`构造函数。

这里的常量变量很好地说明了为什么应该使用变量来为定位器字符串提供上下文。特别是，定位器`tickets-1`和`submit`与它们识别的元素没有任何明显的联系。然而，我们可以通过这些定位器的变量名`ADULT_TICKET_COUNT`和`CHECKOUT_BUTTON`为它们提供一些有意义的上下文:

```
public class CheckoutPage extends BasePage {

  private static final String SECTION_DROP_DOWN_LIST =
  "sectionSelect";
  private static final String ADULT_TICKET_COUNT = "tickets-1";
  private static final String ADD_TICKETS_BUTTON = "add";
  private static final String EMAIL_ADDRESS = "email";
  private static final String CHECKOUT_BUTTON = "submit";

  public CheckoutPage(final AutomatedBrowser automatedBrowser) {
    super(automatedBrowser);
  } 
```

要购买特定区域的门票，我们使用`buySectionTickets()`方法。该方法从下拉列表中选择想要的部分，添加要购买的票的数量，并点击`Add`按钮。

这个动作不会导致任何页面导航，所以我们返回`this`:

```
 public CheckoutPage buySectionTickets(final String section, final Integer adultCount) {

  automatedBrowser.selectOptionByTextFromSelect(section,
  SECTION_DROP_DOWN_LIST, WAIT_TIME);

  automatedBrowser.populateElement(ADULT_TICKET_COUNT,
  adultCount.toString(), WAIT_TIME);

  automatedBrowser.clickElement(ADD_TICKETS_BUTTON, WAIT_TIME);

  return this;
} 
```

我们使用`checkout()`方法购买门票。该方法接受与购买相关联的电子邮件地址，将该电子邮件地址输入适当的字段，并单击`Checkout`按钮。

单击`Checkout`按钮将我们导航到确认页面，因此我们返回一个`ConfirmationPage`类的实例:

```
public ConfirmationPage checkout(final String email) {
  automatedBrowser.populateElement(EMAIL_ADDRESS, email, WAIT_TIME);
  automatedBrowser.clickElement(CHECKOUT_BUTTON, WAIT_TIME);
  return new ConfirmationPage(automatedBrowser);
} 
```

我们来看看`the ConfirmationPage`类:

```
package com.octopus.pages.ticketmonster;

import com.octopus.AutomatedBrowser;
import com.octopus.pages.BasePage;

public class ConfirmationPage extends BasePage {

    private static final String EMAIL_ADDRESS = "div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(2)";
    private static final String EVENT_NAME = "div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(3)";
    private static final String VENUE_NAME = "div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(4)";

    public ConfirmationPage(final AutomatedBrowser automatedBrowser) {
        super(automatedBrowser);
    }

    public String getEmail() {
        return automatedBrowser.getTextFromElement(EMAIL_ADDRESS, WAIT_TIME);
    }

    public String getEvent() {
        return automatedBrowser.getTextFromElement(EVENT_NAME, WAIT_TIME);
    }

    public String getVenue() {
        return automatedBrowser.getTextFromElement(VENUE_NAME, WAIT_TIME);
    }
} 
```

和其他 POM 类一样，这个类扩展了`BasePage`并将`AutomatedBrowser`传递给`BasePage`构造函数。

我们希望在这个页面上与之交互的元素没有可以用来识别它们的属性，迫使我们使用 CSS 选择器。这里常量变量的使用在给这些定位器一些上下文时特别重要，因为像`"div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(2)"`这样的字符串很难破译:

```
public class ConfirmationPage extends BasePage {

    private static final String EMAIL_ADDRESS = "div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(2)";
    private static final String EVENT_NAME = "div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(3)";
    private static final String VENUE_NAME = "div.col-md-6:nth-child(1) > div:nth-child(1) > p:nth-child(4)";

    public ConfirmationPage(final AutomatedBrowser automatedBrowser) {
        super(automatedBrowser);
    } 
```

与其他 POM 类不同，我们没有在这个页面上单击、选择或填充任何元素。然而，我们对从页面中获取一些文本感兴趣，然后我们可以使用这些文本来验证我们购买的门票是否具有正确的值。

这里的 getter 函数返回 3 个段落(或`<p>`)元素的文本内容:

```
public String getEmail() {
    return automatedBrowser.getTextFromElement(EMAIL_ADDRESS, WAIT_TIME);
}

public String getEvent() {
    return automatedBrowser.getTextFromElement(EVENT_NAME, WAIT_TIME);
}

public String getVenue() {
    return automatedBrowser.getTextFromElement(VENUE_NAME, WAIT_TIME);
} 
```

现在让我们来看看使用 POM 类的测试方法:

```
@Test
public void purchaseTicketsPageObjectModel() {

    final AutomatedBrowser automatedBrowser =
            AUTOMATED_BROWSER_FACTORY.getAutomatedBrowser("ChromeNoImplicitWait");

    try {

        automatedBrowser.init();

        final EventsPage eventsPage = new MainPage(automatedBrowser)
                .openPage()
                .buyTickets();

        final VenuePage venuePage = eventsPage
                .selectEvent("Concert", "Rock concert of the decade");

        final CheckoutPage checkoutPage = venuePage
                .selectVenue("Toronto : Roy Thomson Hall")
                .book();

        final ConfirmationPage confirmationPage = checkoutPage
                .buySectionTickets("A - Premier platinum reserve", 2)
                .checkout("email@example.org");

        Assert.assertTrue(confirmationPage.getEmail().contains("email@example.org"));
        Assert.assertTrue(confirmationPage.getEvent().contains("Rock concert of the decade"));
        Assert.assertTrue(confirmationPage.getVenue().contains("Roy Thomson Hall"));

    } finally {
        automatedBrowser.destroy();
    }
} 
```

初始化`AutomatedBrowser`实例的代码与我们之前的测试相同:

```
@Test
public void purchaseTicketsPageObjectModel() {
  final AutomatedBrowser automatedBrowser =
  AUTOMATED_BROWSER_FACTORY.getAutomatedBrowser("ChromeNoImplicitWait");

  try {
    automatedBrowser.init(); 
```

我们的测试从主页面开始，现在由`MainPage`类表示。我们首先创建一个`MainPage`类的新实例，然后链接对`openPage()`和`buyTickets()`方法的调用。

`EventsPage`类的一个实例由`buyTickets()`方法返回。我们将这个值保存在一个名为`eventsPage`的变量中。

请注意，在这段代码中，我们没有引用用于打开页面的 URL，或者用于点击`Buy Tickets Now`链接的定位器。这些细节现在由 POM 类处理，将测试代码从 web 应用程序如何工作的任何特定知识中解放出来:

```
final EventsPage eventsPage = new MainPage(automatedBrowser)
  .openPage()
  .buyTickets(); 
```

浏览场地、结帐和确认页面遵循相同的模式。测试中唯一需要定义的值是音乐会的名称、地点、区域、电子邮件地址和要购买的门票数量。我们没有定义定位符，也没有区分链接和表单按钮:

```
final VenuePage venuePage = eventsPage
  .selectEvent("Concert", "Rock concert of the decade");

final CheckoutPage checkoutPage = venuePage
  .selectVenue("Toronto : Roy Thomson Hall")
  .book();

final ConfirmationPage confirmationPage = checkoutPage
  .buySectionTickets("A - Premier platinum reserve", 2)
  .checkout("email@example.org"); 
```

验证所购门票的详细信息现在也更加简化了。`ConfirmationPage`类通过 getter 方法公开了我们感兴趣的值，测试代码不再需要处理笨拙的 CSS 选择器来查找包含这些信息的段落元素:

```
Assert.assertTrue(confirmationPage.getEmail().contains("email@example.org"));
Assert.assertTrue(confirmationPage.getEvent().contains("Rock concert of the decade"));
Assert.assertTrue(confirmationPage.getVenue().contains("Roy Thomson Hall")); 
```

一旦测试完成，我们就清理`finally`块中的资源:

```
} finally {
  automatedBrowser.destroy();
} 
```

通过使用 POM 设计模式，我们使我们的测试更具可读性，并抽象出了许多与页面(如 URL 或定位器)交互所需的细节，允许针对描述性和流畅的 API 编写测试。

这篇文章是关于[创建 Selenium WebDriver 测试框架](/blog/selenium/0-toc/webdriver-toc)的系列文章的一部分。