---
id: go-execute-activity
title: Executing Activities
---

工作流生效的主要职责是安排要执行的活动。最简单的方法是通过库方法`workflow.ExecuteActivity`。下面的示例代码演示了如何进行此调用：

```go
ao := workflow.ActivityOptions{
        TaskQueue:               "sampleTaskQueue",
        ScheduleToCloseTimeout: time.Second * 60,
        ScheduleToStartTimeout: time.Second * 60,
        StartToCloseTimeout:    time.Second * 60,
        HeartbeatTimeout:       time.Second * 10,
        WaitForCancellation:    false,
}
ctx = workflow.WithActivityOptions(ctx, ao)

var result string
err := workflow.ExecuteActivity(ctx, SimpleActivity, value).Get(ctx, &result)
if err != nil {
        return err
}
```

让我们看一下此调用的每个组成。

##  活动选项

在调用`workflow.ExecuteActivity()`之前，必须配置调用的`ActivityOptions`。这些选项可自定义各种执行超时，并通过从初始 context 创建子 context 并覆盖所需的值来传递。然后将子 context 传递到`workflow.ExecuteActivity()`调用中。如果多个活动共享相同的选项值，则在调用`workflow.ExecuteActivity()`时可以使用相同的 context 实例。

## 活动超时

与活动相关联的超时可能有多种。Temporal 保证活动*最多*执行*一次*，因此活动成功或失败都会伴随以下超时方式之一：

| Timeout                  | **描述**                                                     |
| ------------------------ | ------------------------------------------------------------ |
| `StartToCloseTimeout`    | 工作者收到任务后可以花费的最长时间。                         |
| `ScheduleToStartTimeout` | 工作流安排任务后，活动工作者等待消费任务的时间。如果在指定的时间内没有可用于处理此任务的工作者，则该任务将超时。 |
| `ScheduleToCloseTimeout` | 在工作流程安排任务之后，完成任务所花费的时间。这通常比`StartToClose`和`ScheduleToStart`超时的总和要大。 |
| `HeartbeatTimeout`       | 如果任务在此持续时间内未对 Temporal 服务发出心跳信号，则该任务将被视为失败。这对于长时间运行的任务很有用。 |

## ExecuteActivity 调用

调用中的第一个参数必需是`workflow.Context`对象。这种类型的`context.Context`的`Done()`方法返回`workflow.Channel`，而不是原生 Go `chan`。

第二个参数是我们注册为活动函数的函数。此参数也可以是表示活动函数的完全限定名称的字符串。传递实际函数对象的好处是框架可以验证活动参数。

其余参数作为调用的一部分传递给活动。在我们的示例中，我们有一个参数：`value`。此参数列表必须与活动函数声明的参数列表匹配。Temporal 客户端库将对此进行验证。

本方法的调用会立即返回并返回`workflow.Future`。这使您可以执行更多代码，而不必等待调用的活动完成。

准备好处理活动的结果时，请对返回的 future 对象调用`Get()`方法。该方法的参数是我们传递给`workflow.ExecuteActivity()`调用的`ctx`对象， 以及一个将接收活动输出的出参。输出参数的类型必须与活动函数声明的返回值的类型匹配。该`Get()`方法将阻塞，直到活动完成并且结果可用为止。

您可以检索`workflow.ExecuteActivity()`通过 future 返回的结果值，并像任何同步函数调用中常见的返回结果来使用它。下面的示例代码演示了当返回结果是字符串时是如何使用的：

```go
var result string
if err := future.Get(ctx, &result); err != nil {
        return err
}

switch result {
case "apple":
        // Do something.
case "banana":
        // Do something.
default:
        return err
}
```

在此示例中，我们在`workflow.ExecuteActivity()`之后紧接着在返回的 Future 上调用了`Get()`方法。但是，这不是必需的。如果要并行执行多个活动，则可以重复调用`workflow.ExecuteActivity()`，存储返回的 Future，然后稍后通过调用Future的`Get()`方法来等待所有活动完成。

要在返回的 Future 对象上实现更复杂的等待条件，请使用`workflow.Selector`类。