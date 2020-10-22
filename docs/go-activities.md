---
id: go-activities
title: Activities
---

活动是业务逻辑中特定任务的实现。

活动是通过函数实现的。数据可以通过函数参数直接传递给活动。参数可以是基本类型或结构，唯一的要求是参数必须可序列化。尽管不是必需的，但我们建议活动函数的第一个参数的类型为`context.Context`，以便允许活动与其他框架方法进行交互。该函数必须返回一个`error`值，并且可以选择返回一个结果值。结果值可以是基本类型或结构，唯一的要求是可序列化。

通过调用参数传递到活动或通过结果值返回的值记录在执行历史记录中。整个执行历史记录从 Temporal 服务转移到工作流工作者，其中包含工作流逻辑需要处理的每个事件。因此，大量的执行历史记录可能会对工作流的性能产生不利影响。因此，请注意通过活动调用参数或返回值传输的数据量。除此之外，活动实现不存在其他限制。

## 概述

下面的示例演示一个简单的活动，该活动接受一个字符串参数，向其附加一个单词，然后返回结果。

```go
package sample

import (
	"context"

	"go.uber.org/zap"

	"go.temporal.io/sdk/activity"
)

// SimpleActivity is a sample Temporal activity function that takes one parameter and
// returns a string containing the parameter value.
func SimpleActivity(ctx context.Context, value string) (string, error) {
	activity.GetLogger(ctx).Info("SimpleActivity called.", zap.String("Value", value))
	return "Processed: " + value, nil
}
```
让我们看一下该活动的每个组成部分。

### 声明

在 Temporal 编程模型中，活动是通过函数实现的。函数声明指定活动接受的参数以及活动可能返回的任何值。活动函数可以采用零个或多个活动特定的参数，并且可以返回一个或两个值。它必须始终至少返回一个错误值。活动函数可以将任何可序列化类型作为参数接受，并作为结果返回。

`func SimpleActivity(ctx context.Context, value string) (string, error)`

该函数的第一个参数是 context.Context。这是一个可选参数，可以省略。此参数是标准的 Go context。第二个字符串参数是定制的特定活动参数，可用于在启动时将数据传递到活动中。一个活动可以具有一个或多个这样的参数。活动函数的所有参数必须可序列化，这实际上意味着参数不能是 channels，函数，可变参数或不安全的指针。该活动声明两个返回值：字符串和 error。字符串返回值用于返回活动结果。error 返回值用于指示在执行过程中遇到错误。

### 实现

您可以像其他任何 Go 服务代码一样编写活动实现代码。此外，您可以使用常规的 loggers 和 metrics controllers 以及标准的 Go 并发结构。

#### 心跳

对于长时间运行的活动，Temporal 为活动代码提供了一个 API，可将生存情况和进度报告给 Temporal 托管服务。

```go
progress := 0
for hasWork {
    // Send heartbeat message to the server.
    activity.RecordHeartbeat(ctx, progress)
    // Do some work.
    ...
    progress++
}
```
当一个活动由于丢失心跳而超时，具体实现的最后的值（上面例子中的`progress`）会作为`TimeoutError`的 details 字段从`workflow.ExecuteActivity`函数返回，`TimeoutType` 会被设置为 `Heartbeat`。

您还可以从外部源对活动进行心跳：

```go
// The client is a heavyweight object that should be created once per process.
serviceClient, err := client.NewClient(client.Options{
    HostPort:     HostPort,
    Namespace:   Namespace,
    MetricsScope: scope,
})

// Record heartbeat.
err := serviceClient.RecordActivityHeartbeat(ctx, taskToken, details)
```
该`RecordActivityHeartbeat`函数的参数为：

- `taskToken`：在活动内部检索`TaskToken`的`ActivityInfo`结构的二进制字段的值。
- `details`：包含进度信息的可序列化有效负载。

#### 取消

当活动取消或其工作流执行完成或失败后，传递给其函数的 context 将被取消，这会将其 channel 的关闭状态设置为`Done`。活动可以使用这种方式执行任何必要的清理并中止其执行。取消仅传递给调用了`RecordActivityHeartbeat`的活动。

### 注册

为了使活动对承载它的工作进程可见，必须通过调用`activity.Register`来注册活动。

```go
activity.Register(SimpleActivity)
```
此调用在完全限定的函数名称和实现之间的工作者进程中创建内存映射。如果工作者收到针对其未知活动类型的活动开始执行请求，该请求将失败。

## 标记活动为失败

要将活动标记为失败，活动函数必须通过`error`返回值返回错误。
