---
id: go-signals
title: Signals
---

**信号**提供了一种将数据直接发送到正在运行的工作流的机制。此前，您有两个选项可以将数据传递到工作流实现中：

- 通过启动参数
- 作为活动的返回值

使用启动参数，我们只能在工作流执行开始之前传递值。

活动的返回值使我们能够将信息传递给正在运行的工作流，但是这种方法有其自身的复杂性。一个主要的缺点是依赖轮询。这意味着数据需要存储在第三方位置，直到准备好由活动获取为止。此外，此活动的生命周期需要管理，如果活动在获取数据之前失败，则需要手动重新启动。

而**信号**提供了一种完全异步且持久的机制，用于向运行中的工作流提供数据。当收到正在运行的工作流的信号时，Temporal 将事件和有效负载保留在工作流历史记录中。然后，工作流可以在以后的任何时间处理信号，而不会丢失信息。工作流还可以选择通过阻塞**信号 channel** 来停止执行。

```go
var signalVal string
signalChan := workflow.GetSignalChannel(ctx, signalName)

s := workflow.NewSelector(ctx)
s.AddReceive(signalChan, func(c workflow.Channel, more bool) {
    c.Receive(ctx, &signalVal)
    workflow.GetLogger(ctx).Info("Received signal!", zap.String("signal", signalName), zap.String("value", signalVal))
})
s.Select(ctx)

if len(signalVal) > 0 && signalVal != "SOME_VALUE" {
    return errors.New("signalVal")
}
```

在上面的示例中，工作流代码使用 **workflow.GetSignalChannel** 打开命名的信号的 **workflow.Channel**。然后我们使用 **workflow.Selector** 在此 channel 上等待并处理随信号接收的有效负载。

## SignalWithStart

您可能不知道工作流是否正在运行并且可以接受信号。使用 [client.SignalWithStartWorkflow](https://pkg.go.dev/go.temporal.io/sdk/client#Client) API，可以将信号发送到当前工作流实例（如果存在）或创建新的实例运行后发送信号。因此`SignalWithStartWorkflow`不将运行ID作为参数。 
