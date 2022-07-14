---
layout: post
title: Pulumi 到底比 Terraform 强在哪
category: 技术
---

Pulumi 和 Terraform 都是 Infra-as-Code 领域比较流行的工具，Terraform 出现时间比较早，2014，而 Pulumi 是 2018 年才推出。

这篇总结，我不讲两者的共同之处，只说差异。

作为一个有程序员背景的云计算从业者，我个人更偏爱 Pulumi。Terraform 我也用过，但首先你得学习它自创的 HCL 模板语言，其次如果你的资源定义需要稍微复杂一些的逻辑，比如循环并且有条件的创建一些资源，或者当牵扯一些复杂的条件判断时，模板语言的短板就凸显出来了。

另外一个比较敏感的问题是安全。Pulumi 内置了对敏感信息配置和加密存储包含敏感信息的资源，这些在原生的 Terraform 里是不支持的。

Pulumi 提供了 Automation API 更加方便了你与其他第三方工具集成，比如你自己写的命令行工具或者其他 CI/CD 工具。
![](/images/2022-07-14-pulumi-vs-terraform/pulumi02.jpeg)

如果你是 Terraform 的老手，那么你肯定遇到过有些云服务的功能，在 Terraform 里并不支持，Terraform 的实现是基于这些云平台的 Golang SDK，甚至有些功能在 SDK 里就没有，这个问题在 Terraform 里无解，除非你自己实现或者寄希望于 Terraform 社区慢慢解决。而 Pulumi 推出的 native provider，每天更新与各个云平台的 API 对接，满足你的尝鲜需求。

因为是直接使用编程语言，所以 Pulumi 的强大特性之一就是，你可以在部署资源的前后做一些自定义的操作——任何可以用代码实现的操作，比如部署一个 Helm Chart 时临时修改一个 values 文件不支持的资源属性。你甚至可以直接发送 HTTP 请求到各个云平台做你自己的事情。

最后，如果你来自 Terraform，Pulumi 支持直接将 Terraform 配置文件转换成代码，帮你完成无缝切换。
![](/images/2022-07-14-pulumi-vs-terraform/pulumi01.jpeg)

不可否认，目前在业界，Terraform 比 Pulumi 更加流行，不仅仅是因为 Terraform 比 Pulumi 推出早，其中一个更重要的原因是，使用 Terraform 的很多人都是 Sysadmin 或 Ops 出身，平常工作中顶多写写 bash 脚本，Terraform 对他们来说有着天然的吸引力，在一个文本文件中声明一些资源，然后执行。这一系列操作跟他们之前的工作并没有太大的区别。但 Pulumi 的使用有一些开发门槛，如果想要充分发挥 Pulumi 的优势，就必须至少懂一个它支持的编程语言，比如 Python，TypeScript，Golang 等等。

但是，不要抱怨自己不会编程，毕竟我们生活在 DevOps 的世界里。另外，如果你是管理者，你觉得是寻找懂 HCL 的人容易还是寻找懂 coding 的人容易呢？
