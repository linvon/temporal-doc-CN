---
id: task-queues
title: Task Queues
---

当工作流调用活动时，它将`ScheduleActivityTask` [命令](https://docs.temporal.io/docs/learn-glossary/#command)发送到 Temporal 服务。之后该服务更新工作流的状态，并将[活动任务](https://docs.temporal.io/docs/learn-glossary/#activity-task)分派给实现该活动的 Worker。Temporal 服务使用中间件队列代替直接调用 Worker ，因此会将该服务将*活动任务*添加到此队列，并且 Worker 使用长轮询请求接收该任务。Temporal 称此用于调度活动任务的队列为*活动任务队列*。

同样，当工作流需要处理外部事件时，将创建决策任务。*决策任务队列*用于传送其到工作流 Worker（也称为*决策者*）。

虽然 Temporal 任务队列是队列，但它们与常用的队列技术有所不同。主要的一点是它们不需要显式注册，而是根据需要创建的。任务队列的数量没有限制。一个常见的用例是在每个 Worker 进程中都有一个任务队列，并使用它将活动任务传递给该进程。另一个用例是每个 Worker 池都有一个任务队列。

使用任务队列交付任务而不是通过同步 RPC 调用活动 Worker 有多个优点：

-  Worker 不需要任何开放的端口，更安全。
-  Worker 不需要通过 DNS 或任何其他网络发现机制来公告自己。
- 当所有 Worker 都关闭时，消息将保留在任务队列中，以等待 Worker 恢复。
-  Worker 仅在有可用容量时才轮询消息，因此它永远不会过载。
- 自动平衡大量 Worker 的负载。
- 任务队列支持服务器端限制。这使您可以限制为 Worker 池分配任务的速率 ，并且在出现高峰时仍支持以较高的速率添加任务。
- 任务队列可用于将请求路由到特定的 Worker 池甚至特定的进程。

