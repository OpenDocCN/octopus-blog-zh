# 2.0 中的新功能:轮询触须-章鱼部署

> 原文：<https://octopus.com/blog/new-in-2.0/polling-tentacles>

我们在 Octopus Deploy 2.0 中所做的最复杂的改变之一是为触角引入了**轮询模式**。你可以在文档中了解更多关于[什么是轮询模式，以及如何在轮询模式下设置触手](http://docs.octopusdeploy.com/display/OD/Polling+Tentacles):

> 轮询模式的好处是你不需要在触手端做任何防火墙的改动；你只需要允许访问 Octopus 服务器上的一个端口。

虽然一些部署产品总是以“代理拉”模式工作——代理需要与中央部署服务器直接通信。其他人在“代理监听”模式下工作。Octopus 现在可以在**或者**模式下工作。

![Choosing listen mode](img/2eeb0aa73c9d858cdcf0c5da1110eff1.png)

可以想象，这涉及到内部的一些大的变化。在 1.0 中，每当 Octopus 需要告诉触手做什么事情时，它只会调用一个操作并等待结果。但是当触手在轮询时，那就不行了。相反，我们(我说的我们是指 [Nick](http://octopusdeploy.com/blog/introducing-nick) )建立了一个基于消息的定制通信栈。Octopus 或 Tentacle 可以将发送给对方的消息排队，当连接建立后，通信栈会负责传递消息。

新的通信堆栈使用 SSL 和客户端证书来执行加密，因此无论哪个服务在侦听哪个服务在轮询，它都一样安全(相同的双向信任模型仍然有效)。