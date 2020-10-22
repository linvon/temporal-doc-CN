---
id: go-continue-as-new
title: ContinueAsNew
---

需要定期重新运行的工作流可以原生地实现为带睡眠的大**for**循环，其中工作流的整个逻辑都在**for**循环的主体内。这种方法的问题在于，该工作流的历史记录将持续增长到达到服务所限制的最大大小的程度。

**ContinueAsNew** 是一种底层结构，可以实现这样的工作流而不会出现失败的风险。该操作自动完成当前执行并使用相同的**工作流ID**启动工作流的新执行。新的执行不会继承旧执行的任何历史记录。要触发此行为，工作流函数应通过返回特殊的**ContinueAsNewError**错误来终止

```go
func SimpleWorkflow(workflow.Context ctx, value string) error {
    ...
    return workflow.NewContinueAsNewError(ctx, SimpleWorkflow, value)
}
```
