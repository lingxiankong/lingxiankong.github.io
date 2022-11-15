---
layout: post
title: API Gateway 和 GraphQL Gateway
category: 技术
---

很多人在开发分布式应用时，可能都用过 API Gateway，无论是基于一些开源项目定制还是直接使用云服务，使用 API Gateway 最重要的意义之一是整合，无论后面接入的服务是基于 SOAP 还是 REST，API Gateway 对外提供给用户的是一致的访问体验。其次使用 API Gateway 也是把专业的事情交给专业的服务，跟 API 访问相关的很多功能比如身份认证、流量管理、审计以及 API 的版本管理，都可以放在 API Gateway 上，这样你自己的后台服务就可以专注于自己的业务逻辑。

一些比较出名的 API Gateway 开源实现有：Kong，Gloo，而各个云平台也都提供了自己的 API Gateway 服务，如果你的应用已经云化，那么使用云上的 API Gateway 服务是很自然的事情。

相比于 API Gateway，GraphQL 可能并不为人所熟知。GraphQL Gateway 要解决的问题跟 API Gateway 有重合之处，但它们的使用场景却并不相同。

两者的相同点是都提供 API 访问入口，整合后端不同类型的服务，都提供访问控制、流量监控、审计分析等功能，但不同的是，GraphQL Gateway 可能更适合对接手机 app 或网页的前端应用。比如 GraphQL Gateway 最经典的开发场景是，由前端开发人员定义自己需要的 API 接口，然后交给后端去实现。更方便的是， 后端可以基于前端定义的 API schema 自动生成框架代码，省时省力。使用 GraphQL Gateway 可以减少前端对于服务端 API 的调用次数，并且减少不必要的数据输出，也就是解决了 under/over fetching 的问题。

GraphQL Gateway 也有很流行的开源实现，比如 Apollo，很多开源项目都有对应的商业版，提供更多的高级功能，比如 Apollo Studio，比如 StepZen。与 API Gateway 一样，主流的云平台也都有 GraphQL 服务，比如 AWS AppSync。
