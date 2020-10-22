---
id: go-child-workflows
title: Child Workflows
---

`workflow.ExecuteChildWorkflow` 支持在工作流的实现中编排其他工作流。父工作流具有监视和影响子工作流的生命周期的能力，类似于其对活动调用的方式。

```go
cwo := workflow.ChildWorkflowOptions{
        // Do not specify WorkflowId if you want Temporal to generate a unique Id for the child execution.
        WorkflowId:                   "BID-SIMPLE-CHILD-WORKFLOW",
        ExecutionStartToCloseTimeout: time.Minute * 30,
}
ctx = workflow.WithChildWorkflowOptions(ctx, cwo)

var result string
future := workflow.ExecuteChildWorkflow(ctx, SimpleChildWorkflow, value)
if err := future.Get(ctx, &result); err != nil {
        workflow.GetLogger(ctx).Error("SimpleChildWorkflow failed.", zap.Error(err))
        return err
}
```
让我们看一下此调用的每个组成。

在调用`workflow.ExecuteChildworkflow()`之前，必须配置调用的`ChildWorkflowOptions`。这些选项可自定义各种执行超时，并通过从初始 context 创建子 context 并覆盖所需的值来传递。然后将子 context 传递到`workflow.ExecuteChildWorkflow()`调用中。如果多个活动共享相同的选项值，则在调用`workflow.ExecuteChildworkflow()`时可以使用相同的 context 实例。

调用中的第一个参数必需是`workflow.Context`对象。这种类型的`context.Context`的`Done()`方法返回`workflow.Channel`，而不是原生 Go `chan`。

第二个参数是我们注册为活动函数的函数。此参数也可以是表示活动函数的完全限定名称的字符串。传递实际函数对象的好处是框架可以验证活动参数。

其余参数作为调用的一部分传递到工作流。在我们的示例中，我们有一个参数：`value`。此参数列表必须与工作流函数声明的参数列表匹配。

本方法的调用会立即返回并返回`workflow.Future`。这使您可以执行更多代码，而不必等待调用的工作流完成。

准备好处理工作流的结果时，请对返回的 future 对象调用`Get()`方法。该方法的参数是我们传递给`workflow.ExecuteChildWorkflow()`调用的`ctx`对象， 以及一个将接收工作流输出的输出参数。输出参数的类型必须与工作流函数声明的返回值的类型匹配。该`Get()`方法将一直阻塞，直到工作流完成且结果可用为止。

该`workflow.ExecuteChildWorkflow()`函数类似于`workflow.ExecuteActivity()`。所描述的所有`workflow.ExecuteActivity()`使用模式也都适用于该`workflow.ExecuteChildWorkflow()` 函数。

当用户取消父工作流时，可以根据可配置的子策略取消或放弃子工作流。