---
layout: post
title: Crossplane 是下一代 IaC 么
description: Crossplane 是下一代 IaC 么
category: 技术
---

Crossplane 项目在 2020 年正式进入 CNCF，来自于 Upbound 公司，目前是 CNCF 孵化项目之一。

关于什么是 Crossplane，其官网上用醒目的字体写着：Build cloud native control planes without needing to write code。现在的互联网产品，动不动就强调不用写代码，什么 YAML as Code，Infra as Code，甚至还有 Codeless。净扯淡，代码还在那儿，只不过不让你看见，别人帮你维护。简单的场景还能应付，稍微复杂一些，你就会发现 YAML 不好使了，界面功能也有限，这个时候厂商站出来说，你给我钱，我给你增加功能。你终归是要为代码付费的。Crossplane 里，Provider 就是代码。

Crossplane 是运行在 Kubernetes 集群里的服务。Crossplane 让你能够定义自己的 Kubernetes 风格的 API，达到统一云基础设施和应用程序管理的目的。通过 Crossplane，你可以定义类似于 CRD 的 XRD(CompositeResourceDefinition) 和 XRD 的实现 Composition。比如你定义一个叫 XMyK8SCluster 的 XRD，然后定义三个类似的 Composition 分别对接三大公有云，那么当用户创建一个 MyK8SCluster 资源时，根据指定的 provider，可以在任意一个云平台创建一个 Kubernetes 集群（以及它所有的资源依赖）。这里，XMyK8SCluster 就是你的 API，你向你的用户屏蔽了底层不同云平台之间的实现细节。

这乍一听上去，特别像 Kubernetes Operator 干的事。你定义 CRD 以及实现逻辑 controller，你的用户创建 CR。但 Crossplane 的不同之处在于上面宣传的，你不需要写 Controller（别人给你写好了），只需要定义 Composition。

Crossplane 的故事对于投资人其实很有吸引力，打造一个平台，让任何人基于这个平台定义自己的服务 API，而这个 API 又是 Kubernetes 风格的，意味着你可以复用目前 Kubernetes 生态里的任何工具，比如 Policy Engine，RBAC，Helm，Kustomize，ArgoCD 等等。

很多人刚刚接触 Crossplane 的时候，觉得 Crossplane 是要替代 Terraform，Pulumi 这些 IaC 工具的。因为大部分人看到的 Crossplane 的示例都是基于 AWS 或者 GCP 创建云资源的组合。如果是这样的话，那你的格局就小了。就像它的官网说的，Crossplane 是一个定义 API 的平台，而 Terraform，Pulumi 这些都只是工具，是命令行而已。一个是“平台”，一个是“工具”，气势上能比么？

虽然气势上很足，但 Crossplane 目前有它致命的缺陷。前面提到了，Provider 是 Crossplane 的灵魂，没有 Provider，你在 Crossplane 里定义的那些 XRD，Composition 就是空架子。就以云计算平台为例，截至目前，纵观 Crossplane 里三大云 Provider 定义的 Managed Resource，屈指可数。

Crossplane 的社区规模不大，Upbound 也只是一个创业公司而已，哪有那么些人力去实现各个云平台的 MR？既然是开源，那 Upbound 当然是希望有熟悉三大云的朋友来添砖加瓦，当然，最好是三大云自己的开发人员，毕竟人家是大厂嘛，一两个人随便来参与一下，这个项目就会有质的飞跃。

但三大云又不傻。GCP 人家有自己成熟的 Config Connector，AWS 前两年也发布了自己 AWS Controllers for Kubernetes (ACK)，巴不得大家都锁死在自己的云平台来用自己的服务，谁又会在乎一个潜在的竞争对手呢？

Crossplane 的团队也很聪明，既然没有人做，那就找找其他途径。目前市面上基于不同云平台做事情的工具或服务又不止他们自己一家，别人家是怎么做的呢？Terraform 不就是一个很好的例子嘛。为了集成不同的云平台，Terraform 中也有 Provider 的概念，更重要的是，Terraform 作为目前最为流行的 IaC 工具，它的 Provider 成熟度上比较高的。所以 Crossplane 创建了一个工具，terrajet，基于 Terraform Provider 动态生成 Crossplane Provider，站在巨人的肩膀上，一劳永逸！

所以，Crossplane 真的是下一代 IaC 么？通过上面的 terrajet 项目我们也知道，至少目前，对于云平台能力来说，Crossplane 是受限于 Terraform 的，何谈超越呢？

既然谈到缺陷了，那就再多说一点。目前，Crossplance 在定义 Composition 的时候还是很有局限性的，比如缺乏不同资源间的依赖定义，缺乏相互之间的属性引用，缺乏灵活的属性传递机制等等。这些问题都有折衷的方案，但会让你的 Composition 看起来很复杂。但如果这些都实现了，那么你再看 Composition，就会有一种感觉，它实现了另一套 Terraform。

所以，个人觉得，Crossplance 的故事很好听，但以它目前的能力在实际项目的作用有限，更谈不上生产可用。它背后也只是一家创业公司，如果时运不济，那么这个项目大概率也会消亡。最后，如果你确实感兴趣，条件允许的话，可以考虑做一些贡献，也算是提升个人能力的途径之一吧。
