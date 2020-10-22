---
id: go-versioning
title: Versioning
---

Temporal 使用事件源来重建工作流状态，方法是在工作流定义代码上重放保存的历史事件数据。这意味着，如果处理不当，对工作流程定义代码的任何不兼容更新都可能导致不确定的问题。

## workflow.GetVersion()

考虑以下工作流程定义：

```go
func MyWorkflow(ctx workflow.Context, data string) (string, error) {
        ao := workflow.ActivityOptions{
                ScheduleToStartTimeout: time.Minute,
                StartToCloseTimeout:    time.Minute,
        }
        ctx = workflow.WithActivityOptions(ctx, ao)
        var result1 string
        err := workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, &result1)
        if err != nil {
                return "", err
        }
        var result2 string
        err = workflow.ExecuteActivity(ctx, ActivityB, result1).Get(ctx, &result2)
        return result2, err
}
```
现在，我们已经用 ActivityC 替换了 ActivityA，并部署了更新的代码。如果现在存在一个由工作流原始版本代码启动的工作流执行，其中 ActivityA 已经完成并且结果已记录到历史记录中，新版本的工作流代码将选择该工作流执行并尝试从该处恢复。但是工作流将失败，因为新代码期望从历史数据中获得 ActivityC 的结果，但它将获得 ActivityA 的结果。这将导致工作流因非确定性错误而失败。

因此我们使用  `workflow.GetVersion().`

```go
var err error
v := workflow.GetVersion(ctx, "Step1", workflow.DefaultVersion, 1)
if v == workflow.DefaultVersion {
        err = workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityC, data).Get(ctx, &result1)
}
if err != nil {
        return "", err
}

var result2 string
err = workflow.ExecuteActivity(ctx, ActivityB, result1).Get(ctx, &result2)
return result2, err
```
当`workflow.GetVersion()`为新的工作流程执行运行时，它将在工作流程历史记录中记录一个标记，这样对于此更改Id（示例中的“Step1”）的所有以后对`GetVersion`的调用都将始终返回给定的版本号，在示例中为 `1` 。

如果进行其他更改，例如用 ActivityD 替换 ActivityC，则需要添加一些其他代码：

```go
v := workflow.GetVersion(ctx, "Step1", workflow.DefaultVersion, 2)
if v == workflow.DefaultVersion {
        err = workflow.ExecuteActivity(ctx, ActivityA, data).Get(ctx, &result1)
} else if v == 1 {
        err = workflow.ExecuteActivity(ctx, ActivityC, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
}
```
请注意，我们的`maxSupported`已从1 更改为2。在引入此`GetVersion()`调用之前已经跨过此调用的工作流将返回`DefaultVersion`。在`maxSupported`设置为1的情况下运行的工作流程将返回1。新的工作流程将返回2。

在确定版本1之前的所有工作流程执行均已完成之后，可以删除该版本的代码。现在看起来应该如下所示：

```go
v := workflow.GetVersion(ctx, "Step1", 1, 2)
if v == 1 {
        err = workflow.ExecuteActivity(ctx, ActivityC, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
}
```
您会注意到，`minSupported`已从`DefaultVersion`更改为`1`。如果在此代码上重放了较旧版本的工作流执行历史记录，则将失败，因为最低预期版本为1。确定版本1的所有工作流执行均已完成后，可以删除1，此时代码如下所示：

```go
_ := workflow.GetVersion(ctx, "Step1", 2, 2)
err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
```
请注意，我们保留了对`GetVersion()`的调用。保留此调用有两个原因：

1. 这样可以确保如果仍在运行较旧版本的工作流执行，它将在此处失败并且不会继续。
2. 如果您需要对`Step1`进行其他更改，例如将ActivityD更改为ActivityE，则只需`maxVersion`从2 更新到3并从那里进行分支。

你只需要对每个`changeID`保留第一个`GetVersion()`调用。所有之后的具有相同更改ID的`GetVersion()`调用都可以安全删除。如有必要，您可以删除第一个 `GetVersion()`呼叫，但是需要确保以下几点：

* 使用较旧版本的所有执行均已完成。
* 您不能再使用`Step1`changeId。如果将来需要对同一部分进行更改，例如从ActivityD更改为ActivityE，则需要使用其他的changeId（如）`Step1-fix2`，然后从DefaultVersion重新启动minVersion。该代码如下所示

```go
v := workflow.GetVersion(ctx, "Step1-fix2", workflow.DefaultVersion, 1)
if v == workflow.DefaultVersion {
        err = workflow.ExecuteActivity(ctx, ActivityD, data).Get(ctx, &result1)
} else {
        err = workflow.ExecuteActivity(ctx, ActivityE, data).Get(ctx, &result1)
}
```
如果您不需要保留当前正在运行的工作流程执行，则升级工作流程非常简单。在部署不使用`GetVersion()`的新版本的工作流代码时，您可以简单地终止所有当前正在运行的工作流执行，并暂停创建新的工作流，然后继续创建工作流。但是这不是常见的情况，您需要照顾当前正在运行的工作流执行，因此使用`GetVersion()`更新代码应是您使用的方法。

但是，如果您希望基于当前工作流程逻辑继续当前正在运行的工作流程，但又想确保新工作流程正在新逻辑上运行，则可以将工作流程定义为一个新的 `WorkflowType`，然后将开始路径（调用`StartWorkflow()`）更改为启动新的工作流程类型。

## 健全检查

Temporal 客户端 SDK 会执行完整性检查，以帮助防止明显的不兼容更改。健全性检查以相同的顺序验证重放中的决策是否与历史记录中的事件相匹配。通过调用以下任何方法来生成该决定：

* workflow.ExecuteActivity()
* workflow.ExecuteChildWorkflow()
* workflow.NewTimer()
* workflow.Sleep()
* workflow.SideEffect()
* workflow.RequestCancelWorkflow()
* workflow.SignalExternalWorkflow()

添加，删除或重新排序上述任何方法都会触发完整性检查，并导致不确定性错误。

健全性检查并不执行彻底检查。例如，它不检查活动的输入参数或计时器持续时间。如果对每个属性都执行了检查，那么它将变得过于严格，难以维护工作流代码。例如，如果将活动代码从一个程序包移动到另一个程序包，则会更改`ActivityType`，从技术上讲，这将变成另一种活动。但是，我们不想在这种更改上触发失败，因此我们仅检查 `ActivityType`的函数名称部分。
