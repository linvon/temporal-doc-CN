---
id: system-architecture
title: System architecture
---

## 概述

Temporal 是一个高度可伸缩的，故障无感知的有状态的代码平台。故障无感知的代码是对实现故障容错和持久性的常用技术的进一级抽象。

常见的基于 Temporal 的应用程序由 Temporal 服务，工作流 Worker 和活动 Worker 以及外部客户端组成。请注意，两种类型的 Worker 以及外部客户端都是角色之一，并且可以根据需要在单个应用程序进程中并置它们。

## 临时服务

![时间总览](https://docs.temporal.io/img/docs/learn-topology-overview.png)

Temporal 的核心是高度可扩展的多租户服务。该服务通过强类型的 [Proto API](https://github.com/temporalio/temporal-proto/blob/master/workflowservice/service.proto) 暴露其所有功能。

在内部，它依赖于持久性存储。当前开箱即用地支持 Apache Cassandra 和 MySQL 存储。要使用更复杂的格式列出工作流，可以使用Elasticsearch 集群。 

Temporal 服务负责保持工作流状态和相关的持久化计时器。它维护内部队列（称为任务队列），其用于将任务分派给外部 Worker 。

Temporal 服务是多租户的。因此可以实现不同用例的多个 Worker 池连接到同一服务实例。例如在Uber，一百多个应用程序使用了一项服务。同时，一些外部客户端为每个应用程序部署了一个 Temporal 服务实例。对于本地开发，可以使用通过 docker-compose 配置的本地 Temporal 服务实例。

![时间总览](https://docs.temporal.io/img/docs/temporal-overview.svg)

## 工作流  Worker 

Temporal 重用*工作流自动化*命名空间中的术语。因此故障无感知的有状态代码称为工作流。

Temporal 服务不会直接执行工作流代码。工作流代码由外部（从服务的角度）*工作流 Worker*进程托管。这些流程从 Temporal 服务中接收*决策任务*，这些*任务*包含工作流需要处理的事件，并将其传递给工作流代码，最终将工作流*决策*传达回服务。

由于工作流代码在服务外部，因此可以用可以与服务 Thrift API 进行通讯的任何语言来实现。目前，Java 和 Go 客户端已准备就绪。Python 和 C＃客户端正在开发中。如果您有兴趣用您的首选语言为客户端提供贡献，请告诉我们。

Temporal 服务 API 并不强加任何特定的工作流定义语言。因此特定的 Worker 可以被实现用来执行几乎所有现有的工作流规范。Temporal 团队选择的开箱即用的模型基于持久化功能的思想。持久化的功能尽可能地接近应用程序业务逻辑，而只需要的很少的衔接代码。

## 活动 Worker 

工作流故障无感知的代码不受基础架构故障的影响。但是它必须与不完美的外部世界进行沟通，在那里失败是很常见的。所有与外界的交互都是通过活动进行的。活动是代码段，可以执行任何特定于应用程序的操作，例如调用服务、更新数据库记录或从 Amazon S3 下载文件。与队列系统相比，Temporal 活动的功能非常丰富。示例特性包括将任务路由到特定进程、无限重试、心跳和无限执行时间。

活动由*活动 Worker*进程主持，这些*活动 Worker*进程从 Temporal 服务接收*活动任务*，调用相应的活动实现并报告任务完成状态。

## 外部客户端

工作流和活动 Worker 托管工作流和活动代码。但是要创建工作流实例（使用时间术语执行）时应调用 Temporal 服务的 `StartWorkflowExecution` API。通常工作流由外部实体（如UI、微服务或 CLI）启动。

这些实体还可以：

- 以信号形式通知工作流有关异步外部事件的信息
- 同步查询工作流状态
- 同步等待工作流完成
- 取消、终止、重新启动和重置工作流
- 使用列表 API 搜索特定的工作流