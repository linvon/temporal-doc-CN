---
id: go-create-workflows
title: Creating Workflows
---

工作流是编排逻辑的实现。Temporal 编程框架（也称为客户端库）使您可以将工作流编排逻辑编写为使用标准 Go 数据建模的简单过程代码。客户端库负责处理 Worker 服务和 Temporal 服务之间的通信，并确保事件之间的状态持久性，即使在 Worker 失败的情况下也是如此。此外，任何特定的执行都不会与特定的 Worker 相关联。编排逻辑的不同步骤最终可能在不同的 Worker 实例上执行，而框架确保在执行该步骤的 Worker 上重新创建必要的状态。

但是，为了促进这种操作模型，Temporal 编程框架和托管服务都对编排逻辑的实施施加了一些要求和限制。这些要求和限制的详细信息在下面的“ **实现”**部分中进行了描述 。

## 概述

下面的示例代码展示了执行一个活动的工作流的简单实现。工作流还将作为初始化的一部分接收的唯一参数作为参数传递给活动。

```go
package sample

import (
	"time"

	"go.temporal.io/sdk/workflow"
	"go.uber.org/zap"
)

// SimpleWorkflow is a sample Temporal workflow function that takes one
// string parameter 'value' and returns an error.
func SimpleWorkflow(ctx workflow.Context, value string) error {
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

	workflow.GetLogger(ctx).Info("Done", zap.String("result", result))
	return nil
}
```

## 声明

在 Temporal 编程模型中，工作流通过函数来实现。函数声明指定工作流接受的参数以及它可能返回的任何值。

```go
    func SimpleWorkflow(ctx workflow.Context, value string) error
```

让我们解构上面的声明：

- 该函数的第一个参数是 **ctx working.Context**。这是所有工作流功能的必需参数，并且由 Temporal 客户端库用于传递执行 context 。几乎所有可从工作流功能调用的客户端库功能都需要此 **ctx** 参数。该 **context** 参数与 Go 提供的标准的 **context.Context** 概念相同 。**workflow.Context** 和 **context.Context**之间的唯一区别是 **workflow.Context ** 的 **Done()** 函数返回 **workflow.Channel** 而不是标准的 go **chan**。
- 第二个参数 **string** 是自定义工作流参数，可用于在启动时将数据传递到工作流中。工作流可以具有一个或多个这样的参数。工作流函数的所有参数必须可序列化，这实际上意味着参数不能是 channel、函数、可变参数或不安全的指针。
- 由于仅将错误声明为返回值，因此这意味着工作流不会返回其他数据值。**错误**返回值被用于指示执行过程中遇到一个错误，该工作流应该被终止。

## 实现

为了支持工作流实现的同步和顺序编程模型，必须在工作流实现如何呈现上存在某些限制和要求以确保正确性。要求是：

- 执行必须是确定性的
- 执行必须是幂等的

满足这些要求的一种直接方法是工作流代码应如下：

- 工作流代码只能读取和操作本地状态或从 Temporal 客户端库函数作为返回值接收的状态。
- 除了通过活动调用之外，工作流代码不应与外部系统交互。
- 工作流代码应仅通过 Temporal 客户端库提供的功能与**时间**进行交互**（例如**，**workflow.Now()**，**workflow.Sleep()**）。
- 工作流代码不应直接创建 goroutine 并与 goroutines 进行交互，而应使用 Temporal 客户端库提供的功能（例如，**workflow.Go()** 代替 **go**， **workflow.Channel **而不是 **chan**，**workflow.Selector** 而不是 **select**）。
- 工作流代码应通过 Temporal 客户端库提供的日志器（即 **workflow.GetLogger()**）进行所有日志行为。
- 工作流代码不应使用 range 在 maps 上进行迭代，因为 maps 迭代的顺序是随机的。

现在，我们已经奠定了基本规则，我们可以看看用于编写 Temporal 工作流的一些特殊功能和类型，以及如何实现一些通用模式。

### 特殊 Temporal SDK 函数和类型

Temporal 客户端库提供了许多函数和类型以替代某些本地 Go 函数和类型。为了确保工作流代码执行在执行 context 中具有确定性和幂等性，必须使用这些替换掉相关函数和类型。

协程相关的构造：

- **workflow.Go**：这是 **go** 语句的替代。
- **workflow.Channel**：这是原生 **chan** 类型的替代。Temporal 同样提供缓冲通道和非缓冲通道支持。
- **stream.Selector**：这是 **select** 语句的替代。

时间相关功能：

- **workflow.Now()** ：这是 **time.Now()** 的替代。
- **workflow.Sleep()**：这是 **time.Sleep()** 的替代。

### 标记一个工作流失败

要将工作流标记为失败，所有要做的事情是使工作流函数通过 **err** 返回值返回错误。

## 注册

为了使某些客户端代码能够调用工作流， Worker 进程需要知道其有权访问的所有实现。通过以下调用来注册工作流：

```go
workflow.Register(SimpleWorkflow)
```

此调用实质上是在完全限定的函数名称和实现之间的 Worker 进程中创建了内存映射。从 **init()**函数调用此注册方法是安全的。如果 Worker 收到未知的工作流类型的任务，它将使该任务失败。但是任务失败不会导致整个工作流失败。