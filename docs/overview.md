---
id: overview
title: Overview
sidebar_label: Overview
description: This guide will help you build your own resilient applications using Temporal Workflow as Code™
---

大多数实际用例场景远不止一个请求响应体那么简单，它们需要跟踪复杂的状态、响应异步事件，并与外部不可靠的依赖项进行通信。构建此类应用程序的常用方法是对无状态服务，数据库，定时任务和队列系统等进行大杂烩。因为充斥着大量代码用于衔接，从而掩盖了大量底层细节背后的实际业务逻辑，这对开发人员的生产效率会产生负面影响。同时由于很难使所有组件保持健康，因此此类系统经常会出现可用性问题。

Temporal 解决方案是一种[*故障无感知的有状态的*编程模型](https://docs.temporal.io/docs/workflows/)，可以消除绝大多数在构建可扩展分布式应用程序时的复杂性。本质上，Temporal 提供了未链接到特定进程的持久虚拟内存，并保留了完整的应用程序状态（包括函数堆栈）以及跨各种主机和软件故障的局部变量。这使您可以发挥一个编程语言的全部功力来编写代码，而 Temporal 则来负责应用程序的耐用性，可用性和可扩展性。

Temporal 由编程框架（客户端库）和托管服务（后端）组成。框架可以使开发人员能够以熟悉的语言编写和编排任务（ [Go](https://github.com/temporalio/temporal-go-sdk/) 和 [Java](https://github.com/temporalio/temporal-java-sdk) 如今已经支持，在 [Python](https://github.com/firdaus/cadence-python) 和  [C＃](https://github.com/nforgeio/neonKUBE/tree/master/Lib/Neon.Cadence) 中通过[代理](https://github.com/nforgeio/neonKUBE/tree/master/Go/src/github.com/loopieio/cadence-proxy)来运行的某些项目正在开发中）。

该框架使开发人员可以用熟悉的语言编写故障无感知的代码。（[Go](https://github.com/temporalio/temporal-go-sdk/) 和 [Java](https://github.com/temporalio/temporal-java-sdk) 可以用于生产中。[Python](https://github.com/firdaus/cadence-python) 和 [C＃](https://github.com/nforgeio/neonKUBE/tree/master/Lib/Neon.Cadence) 正在开发中）。

后端服务是无状态的，并且依赖于持久性存储。当前支持 Cassandra 和 MySQL 存储方式。实际上任何其他支持多行单分片事务的数据库都可以被添加支持。服务有不同的部署模型。在 Uber，我们的团队运营着由数百个应用程序共享的多租户集群。

观看 Uber Open Summit 上 Maxim 的演讲，了解有关 Temporal 编程模型和价值主张的介绍。

[![](https://res.cloudinary.com/marcomontalbano/image/upload/v1603718168/video_to_markdown/images/youtube--llmsBGKOuWI-c05b58ac6eb4c4700831b2b3070cd403.jpg)](https://www.youtube.com/embed/llmsBGKOuWI "")

Temporal 服务的 GitHub 存储库是 [temporalio/temporal](https://github.com/temporalio/temporal)。Temporal 服务的 Docker 镜像可以再在 Docker Hub 上的[temporalio/server](https://hub.docker.com/r/temporalio/server) 上获取 。

