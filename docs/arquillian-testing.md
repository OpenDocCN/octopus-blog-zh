# 阿奎利亚测试介绍- Octopus 部署

> 原文：<https://octopus.com/blog/arquillian-testing>

让我们用两个类创建一个简单的 EJB 应用程序。第一种称为`EnterpriseJavaBean`，向控制台写入一个 UUID，休眠一段时间，然后再次向控制台写入同一个 UUID。

```
package org.example.arquilliantest;

import javax.ejb.Asynchronous;
import javax.ejb.Singleton;
import java.util.Random;
import java.util.UUID;

@Singleton
public class EnterpriseJavaBean {
    @Asynchronous
    public void writeToConsole() {
        final UUID uuid = UUID.randomUUID();
        System.out.println(uuid.toString());

        try {
            Thread.sleep(Math.abs(new Random().nextLong()) % 100);
        } catch (InterruptedException e) {
            // ignored
        }

        System.out.println(uuid.toString());
    }
} 
```

第二个类叫做`StartupService`，在启动时构造，构造后调用`EnterpriseJavaBean.writeToConsole()` 10 次。

```
package org.example.arquilliantest;

import javax.annotation.PostConstruct;
import javax.ejb.EJB;
import javax.ejb.Singleton;
import javax.ejb.Startup;

@Startup
@Singleton
public class StartupService {
    @EJB
    private EnterpriseJavaBean enterpriseJavaBean;

    @PostConstruct
    public void postConstruct() {
        for (int i = 0; i < 10; ++i) {
            enterpriseJavaBean.writeToConsole();
        }
        System.out.println("All Done");
    }
} 
```

这里的代码是米尔 EJB 逻辑的运行，你可以在任何 Java EE 应用程序中找到。一旦编译并运行，类似下面的内容将被打印到控制台。

```
12:56:19,359 INFO  [stdout] (EJB default - 4) 2a3c2709-c556-49be-8ca7-25778b54310d
12:56:19,442 INFO  [stdout] (EJB default - 4) 2a3c2709-c556-49be-8ca7-25778b54310d
12:56:19,444 INFO  [stdout] (EJB default - 9) 4be307b4-9e4b-4f37-b1b4-a54f234a591c
12:56:19,471 INFO  [stdout] (EJB default - 9) 4be307b4-9e4b-4f37-b1b4-a54f234a591c
12:56:19,472 INFO  [stdout] (EJB default - 10) ef7a63ae-1e65-4d9e-aed0-97ddf6681956
12:56:19,519 INFO  [stdout] (EJB default - 10) ef7a63ae-1e65-4d9e-aed0-97ddf6681956
12:56:19,520 INFO  [stdout] (EJB default - 2) 8dc989b8-48e2-4128-9b3a-d80a883033a7
12:56:19,615 INFO  [stdout] (EJB default - 2) 8dc989b8-48e2-4128-9b3a-d80a883033a7
12:56:19,618 INFO  [stdout] (EJB default - 1) 4ae8e179-6b3c-44c3-9cee-62a93c069bba
12:56:19,719 INFO  [stdout] (EJB default - 1) 4ae8e179-6b3c-44c3-9cee-62a93c069bba
12:56:19,721 INFO  [stdout] (EJB default - 6) 3cf93b22-dd92-45ea-95d3-73873322579e
12:56:19,784 INFO  [stdout] (EJB default - 6) 3cf93b22-dd92-45ea-95d3-73873322579e
12:56:19,786 INFO  [stdout] (EJB default - 3) b38ce136-78dc-49b5-85db-c11f491a5f14
12:56:19,856 INFO  [stdout] (default task-1) All Done
12:56:19,877 INFO  [stdout] (EJB default - 3) b38ce136-78dc-49b5-85db-c11f491a5f14
12:56:19,878 INFO  [stdout] (EJB default - 7) f0bb9c74-fdec-4994-b66d-296e310da24d
12:56:19,945 INFO  [stdout] (EJB default - 7) f0bb9c74-fdec-4994-b66d-296e310da24d
12:56:19,946 INFO  [stdout] (EJB default - 5) e8ef8311-dd2d-45c1-af07-a3059d92f680
12:56:19,972 INFO  [stdout] (EJB default - 5) e8ef8311-dd2d-45c1-af07-a3059d92f680
12:56:19,973 INFO  [stdout] (EJB default - 8) b40cf503-c05c-4282-95be-3bf03875bc2f
12:56:20,045 INFO  [stdout] (EJB default - 8) b40cf503-c05c-4282-95be-3bf03875bc2f
12:56:20,046 INFO  [stdout] (EJB default - 4) a0a70b9f-dd13-41fd-95a9-f8b38a062377
12:56:20,085 INFO  [stdout] (EJB default - 4) a0a70b9f-dd13-41fd-95a9-f8b38a062377
12:56:20,086 INFO  [stdout] (EJB default - 9) f8069c90-c980-4593-805f-352c7fcb13ae
12:56:20,110 INFO  [stdout] (EJB default - 9) f8069c90-c980-4593-805f-352c7fcb13ae
12:56:20,111 INFO  [stdout] (EJB default - 10) e0fe386b-18e5-4883-a14f-a8af75b010ef
12:56:20,116 INFO  [stdout] (EJB default - 10) e0fe386b-18e5-4883-a14f-a8af75b010ef
12:56:20,118 INFO  [stdout] (EJB default - 2) 9d057b61-cb3f-4da4-b401-9f1e8d251669
12:56:20,215 INFO  [stdout] (EJB default - 2) 9d057b61-cb3f-4da4-b401-9f1e8d251669
12:56:20,217 INFO  [stdout] (EJB default - 1) 17b80b1e-6885-4a38-8051-7246854fd54c
12:56:20,227 INFO  [stdout] (EJB default - 1) 17b80b1e-6885-4a38-8051-7246854fd54c
12:56:20,229 INFO  [stdout] (EJB default - 6) 81dcd134-e704-4c77-b8b0-04a15e163705
12:56:20,302 INFO  [stdout] (EJB default - 6) 81dcd134-e704-4c77-b8b0-04a15e163705
12:56:20,303 INFO  [stdout] (EJB default - 3) 4e32ecb9-9e95-42b0-8077-3cb80f40faca
12:56:20,398 INFO  [stdout] (EJB default - 3) 4e32ecb9-9e95-42b0-8077-3cb80f40faca 
```

让我们假设上面的输出序列是有效的输出，并且我们想在单元测试中验证这个行为。

## 用 JUnit 测试 POJOs

EJB 3 规范的卖点之一是它允许你编写 POJOs。通过一些额外的注释，这些 POJOs 变成了完全成熟的 EJB。或者至少它们会在合适的环境中成为 EJB。但稍后会详细介绍。

因为这些类是 POJOs，我们可以很容易地将它们合并到单元测试中。

忽略实际捕获和验证控制台输出所需的工作，这就是我们的测试可能的样子。它复制了在`StartupService`类中找到的相同逻辑，所以您可能认为它会产生几乎相同的输出。

```
@Test
public void plainTest() {
    final EnterpriseJavaBean enterpriseJavaBean = new EnterpriseJavaBean();
    for (int i = 0; i < 10; ++i) {
        enterpriseJavaBean.writeToConsole();
    }
    System.out.println("All Done");
} 
```

当测试运行时，我们接近原始输出。

```
dc1d071a-18f9-4cd4-9f90-70768c818165
dc1d071a-18f9-4cd4-9f90-70768c818165
983ade5f-f514-4530-bd3e-e5c71910b3b4
983ade5f-f514-4530-bd3e-e5c71910b3b4
1ce137d0-1d9e-4f26-9b00-75ae72fb4cee
1ce137d0-1d9e-4f26-9b00-75ae72fb4cee
cd8bc751-6021-4a92-ab0a-03021f66c353
cd8bc751-6021-4a92-ab0a-03021f66c353
916e331c-2b54-4c9c-b3e6-f2625d816f7b
916e331c-2b54-4c9c-b3e6-f2625d816f7b
706bf787-d35d-49f2-8963-39010b53245d
706bf787-d35d-49f2-8963-39010b53245d
afa3d953-0599-4f8d-8e58-c22f79a7c789
afa3d953-0599-4f8d-8e58-c22f79a7c789
9ac2dc9b-dc8c-4f96-ba98-f38b299d9be8
9ac2dc9b-dc8c-4f96-ba98-f38b299d9be8
07e9bf94-6e38-4420-a175-81f436451832
07e9bf94-6e38-4420-a175-81f436451832
9027ca83-690d-4d45-beda-4ae599844f44
9027ca83-690d-4d45-beda-4ae599844f44
All Done 
```

敏锐的观察者会注意到,`All Done`消息总是打印在单元测试输出的末尾，但是在部署到服务器时，在执行`EnterpriseJavaBean`类的过程中随机打印。

这是因为`writeToConsole()`方法被标记为`@Asynchronous`。

这些 EJB 注释会被任何不能识别它们的代码忽略，我们的 JUnit 测试就是一个不能识别 EJB 注释的执行环境的例子。这意味着单元测试将同步调用这个方法，这又意味着`All Done`消息将总是最后显示。

这第一次表明测试 EJB 并不像看起来那样简单。但到目前为止，差异是显而易见的；我们正在测试的方法被清楚地标记为`@Asynchronous`，所以我们可以很容易地复制这个行为。让我们在执行器内部调用`writeToConsole()`方法，它将以异步方式进行调用。

```
@Test
public void plainThreadTest() throws InterruptedException {
    final EnterpriseJavaBean enterpriseJavaBean = new EnterpriseJavaBean();

    try {
        final ExecutorService executor = Executors.newFixedThreadPool(10);

        final List<Callable<Void>> tasks = new ArrayList<>();
        for (int i = 0; i < 10; ++i) {
            executor.submit(() -> {
                enterpriseJavaBean.writeToConsole();
                return null;
            });
        }

        System.out.println("All Done");
        executor.shutdown();
        executor.awaitTermination(10, TimeUnit.SECONDS);
    } catch (Exception e) {
        // ignored
    }
} 
```

这会产生以下输出:

```
All Done
b887112d-d92e-4a1c-8986-0b1e25916c99
4ded12d5-a837-495a-a199-059cb58a2f11
63e12097-6c62-416c-ae60-bd873554e376
18a722fe-548a-48c1-bc6c-eaec651522fa
8e2273c0-73cc-4dfc-9964-a1c6c40055b4
c41acb02-61fc-4a4b-bd68-45545b4ed734
b74acccf-1b38-45c8-9375-1d7380698214
85b71c13-9b3c-4e4b-bdb7-32064a6a9818
c2fa0677-291a-4066-8dcb-13f4a6896489
3ed6dc3a-590e-4350-a6a4-fc0e3e799505
b74acccf-1b38-45c8-9375-1d7380698214
4ded12d5-a837-495a-a199-059cb58a2f11
18a722fe-548a-48c1-bc6c-eaec651522fa
85b71c13-9b3c-4e4b-bdb7-32064a6a9818
63e12097-6c62-416c-ae60-bd873554e376
c41acb02-61fc-4a4b-bd68-45545b4ed734
b887112d-d92e-4a1c-8986-0b1e25916c99
c2fa0677-291a-4066-8dcb-13f4a6896489
8e2273c0-73cc-4dfc-9964-a1c6c40055b4
3ed6dc3a-590e-4350-a6a4-fc0e3e799505 
```

现在我们从线程池中调用`writeToConsole()`方法，这让我们更接近 EJB 调用`@Asynchronous`方法的方式。`All Done`消息现在不再总是在输出的末尾，但是 UUIDs 都混在一起了。这不是我们在服务器上执行`writeToConsole()`时看到的行为，它总是一个接一个地打印匹配的 UUID 对。

这里的问题是，当`EnterpriseJavaBean`类上的方法被作为 EJB 注入时被调用，这些方法默认为由`@Lock(WRITE)`注释定义的语义。这意味着一次只能调用一个方法，这确保了 UUID 对总是一个接一个地被打印。

但是当您查看`EnterpriseJavaBean`类的代码时，这种行为并不明显。任何地方都没有`@Lock(WRITE)`标注；这是 EJB 容器采用的默认值。

然后，我们可以继续尝试围绕`writeToConsole()`方法添加一些同步，但是此时我们已经花费了更多的时间来复制执行 EJB 的环境，而不是验证这些方法中的业务逻辑。

## 阿奎利亚人请到回避区

Arquillian 项目解决了这种情况。Arquillian 允许您利用服务器环境赋予对象的任何功能，针对 EJB(或任何类型的增强对象)编写测试。

编写一个 Arquillian 测试实际上很容易。下面是一个复制了`EnterpriseJavaBean`类行为的例子。

```
package org.example.arquilliantest;

import org.jboss.arquillian.container.test.api.Deployment;
import org.jboss.arquillian.junit.Arquillian;
import org.jboss.shrinkwrap.api.ShrinkWrap;
import org.jboss.shrinkwrap.api.spec.JavaArchive;
import org.junit.Test;
import org.junit.runner.RunWith;

import javax.ejb.EJB;

@RunWith(Arquillian.class)
public class ArquillianTest {

    @EJB
    private EnterpriseJavaBean enterpriseJavaBean;

    @Deployment
    public static JavaArchive createDeployment() {
        JavaArchive jar = ShrinkWrap.create(JavaArchive.class)
                .addClass(EnterpriseJavaBean.class);
        return jar;
    }

    @Test
    public void plainThreadTest() throws InterruptedException {
        for (int i = 0; i < 10; ++i) {
            enterpriseJavaBean.writeToConsole();
        }
        System.out.println("All Done");
    }
} 
```

这个测试有几个重要的方面。

`@RunWith(Arquillian.class)`注释用 Arquillian 提供的功能丰富了 JUnit 测试。

带有`@Deployment`注释的`createDeployment()`方法用于创建工件，该工件被部署为测试环境。在本例中，我们正在构建一个包含`EnterpriseJavaBean`类的 jar 文件。这是使用 [ShrinkWrap](http://arquillian.org/guides/shrinkwrap_introduction/) 库完成的，它允许我们用代码创建 Java 工件，就像 Maven 这样的构建工具在构建时所做的一样。

在测试中，我们使用`@EJB`注释注入了一个`EnterpriseJavaBean`类的实例。当测试运行时，这个变量引用的对象是一个真实的、活的 EJB。这意味着，我们不是测试一个带有被忽略的 EJB 注释的 POJO(因此测试一个没有 EJB 功能的对象)，而是测试一个行为方式与部署到应用服务器时相同的对象。

最后,`@Test`方法本身运行我们最初试图编写的相同测试代码，但是这一次的输出正是我们在将代码部署到真实服务器时所看到的。

## 结论

通过使用 Arquillian 运行测试，可以测试一个对象在部署到应用服务器时所提供的实际功能。这使您不必试图复制应用服务器赋予的功能，并且意味着您的测试反映了您的生产代码。

你可以从 GitHub 获得这个项目的源代码。

如果您对 Java 应用程序的自动化部署感兴趣，[下载 Octopus Deploy](https://octopus.com/downloads) 的试用版，并查看我们的文档。