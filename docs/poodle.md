# SSL 3.0“狮子狗”和 Octopus 部署- Octopus 部署

> 原文：<https://octopus.com/blog/poodle>

有一个新发现的名为 [POODLE](https://www.openssl.org/~bodo/ssl-poodle.pdf) 的安全漏洞:

> 上述攻击需要建立 SSL 3.0 连接，因此在客户端或服务器(或两者)中禁用 SSL 3.0 协议可以完全避免这种攻击。如果任何一方只支持 SSL 3.0，那么所有的希望都没有了，需要一个严肃的更新来避免不安全的加密。如果 SSL 3.0 既没有被禁用，也不是唯一可能的协议版本，那么如果客户端使用降级舞蹈来实现互操作性，则攻击是可能的。

正如我们在 [Heartbleed 和 Octopus Deploy](https://octopusdeploy.com/blog/heartbleed) 的帖子中所讨论的，我们使用。NET framework 的`SslStream`类，用于在 Octopus 部署服务器和触手部署代理通信时建立安全连接。

创建`SslStream`时，指定要使用的协议。[。NET 4.0 支持 SSL 2.0、3.0 和 TLS 1.0](http://msdn.microsoft.com/en-us/library/system.security.authentication.sslprotocols(v=vs.100).aspx) 。[。NET 4.5 支持 SSL 2.0、3.0 和 TLS 1.0、1.1 和 1.2](http://msdn.microsoft.com/en-us/library/system.security.authentication.sslprotocols(v=vs.110).aspx) 。

有趣的是，默认的协议值(在两者中。NET 4.0 和 4.5)是`Tls | Ssl3`。换句话说，TLS 1.0 是首选，但是如果客户机/服务器只支持 SSL 3.0，那么它将会退回到那个版本。正如本文所讨论的，即使您的客户机/服务器支持 TLS，这也是一个问题，因为攻击者可以强制降级。

但是有一个好消息——在 Octopus 中，当我们构造我们的`SslStream`时，我们指定了要使用的协议—**,我们特别限制了与 TLS 1.0** 的连接(Octopus 运行于。NET 4.0 所以我们还做不到 TLS 1.1/1.2)。由于我们同时控制了客户端和服务器，所以我们不需要担心退回到 SSL 3.0，所以我们不允许这样做。

我们实际上已经做了很长时间了。2013 年 1 月，我们发布了一个名为 [Halibut](https://github.com/OctopusDeploy/Halibut) 的开源项目，这是一个原型，最终演变成了我们在章鱼和触手之间使用的通信堆栈。即使在那时，我们也明确表示只支持 TLS:

```
ssl.AuthenticateAsServer(serverCertificate, true, SslProtocols.Tls, false); 
```

Octopus web 门户(用于管理 Octopus 服务器的 HTML web 前端)的情况略有不同。门户托管在 HTTP.sys 之上，http . sys 是 IIS 背后的内核模式驱动程序。开箱即用的门户使用 HTTP，但是如果您喜欢的话，您可以[将您的 web 门户配置为通过 HTTPS 可用。](http://docs.octopusdeploy.com/display/OD/Expose+the+Octopus+web+portal+over+HTTPS)

据我所知，IIS 和 HTTP.sys 使用 SChannel 支持的任何协议，这意味着它们将允许 SSL 3.0。看起来需要[改变注册表](https://www.digicert.com/ssl-support/iis-disabling-ssl-v3.htm)来禁用 SChannel 中的 SSL 3.0，以防止 IIS/HTTP.sys 使用它。

微软也有一个[安全顾问](https://technet.microsoft.com/en-us/library/security/3009008.aspx)，使用组策略来禁用 SSL 3.0，但它似乎专注于 Internet Explorer 而不是 IIS。

**TL；章鱼/触手的交流不受狮子狗的影响。门户网站(如果你在 HTTPS 公开它)可能是，就像你在 HTTPS 使用 IIS 服务的任何其他网站可能是。**

和往常一样，特洛伊·亨特对狮子狗臭虫有一篇很好的报道。