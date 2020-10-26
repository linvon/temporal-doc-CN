---
id: go-distributed-cron
title: Distributed CRON
---

将任何 Temporal 工作流转换为 Cron 工作流非常简单。您需要做的就是在启动工作流时使用 [StartWorkflowOptions](https://pkg.go.dev/go.temporal.io/sdk/internal#StartWorkflowOptions) 的 CronSchedule 参数提供 cron 计划 。

您还可以使用 Temporal CLI 启动工作流，并使用可选的`--cron`参数使用 cron 作业。 

对于使用 CronSchedule 的工作流：

* Cron 作业基于 UTC 时间。例如，cron 作业“ 15 8 * * * ”将每天在世界标准时间上午8:15运行。
* 如果工作流失败并且 StartWorkflowOptions 也提供了 RetryPolicy，则工作流将基于 RetryPolicy 重试。重试工作流时，服务器不会安排下一次 cron 运行。
* Temporal 服务仅在当前运行完成后才调度下一个 cron 运行。如果在工作流正在运行（或重试）时下一个计划到期，则它将跳过该计划。
* Cron 工作流直到终止或取消后才会停止。

Temporal 支持标准 cron 规范:

```go
// CronSchedule - Optional cron schedule for workflow. If a cron schedule is specified, the workflow will run
// as a cron based on the schedule. The scheduling will be based on UTC time. The schedule for next run only happen
// after the current run is completed/failed/timeout. If a RetryPolicy is also supplied, and the workflow failed
// or timed out, the workflow will be retried based on the retry policy. While the workflow is retrying, it won't
// schedule its next run. If next schedule is due while the workflow is running (or retrying), then it will skip that
// schedule. Cron workflow will not stop until it is terminated or cancelled (by returning temporal.CanceledError).
// The cron spec is as following:
// ┌───────────── minute (0 - 59)
// │ ┌───────────── hour (0 - 23)
// │ │ ┌───────────── day of the month (1 - 31)
// │ │ │ ┌───────────── month (1 - 12)
// │ │ │ │ ┌───────────── day of the week (0 - 6) (Sunday to Saturday)
// │ │ │ │ │
// │ │ │ │ │
// * * * * *
CronSchedule string
```

该 [crontab guru site](https://crontab.guru/) 网站可以检验你的 cron 表达式是否是有效的。

## 转换现有的 cron 工作流

在可使用 CronSchedule 之前，实现 cron 工作流的老方法是将延迟计时器用作最后一步，然后返回 `ContinueAsNew`。该实现的一个问题是，如果工作流失败或超时，则 cron 将停止。

要将这些工作流转换为使用 Temporal CronSchedule，您所需要做的就是删除延迟计时器，然后不使用而返回 `ContinueAsNew`。然后使用所需的CronSchedule 启动工作流。 

## 检索最后的成功结果

有时，获取先前成功运行的进度很有用。
这在客户端库中有两个新的 API 可以支持：
`HasLastCompletionResult` and `GetLastCompletionResult`. 

以下是如何在Go中使用此方法的示例：

```go
func CronWorkflow(ctx workflow.Context) (CronResult, error) {
    startTimestamp := time.Time{} // By default start from 0 time.
    if workflow.HasLastCompletionResult(ctx) {
        var progress CronResult
        if err := workflow.GetLastCompletionResult(ctx, &progress); err == nil {
            startTimestamp = progress.LastSyncTimestamp
        }
    }
    endTimestamp := workflow.Now(ctx)

    // Process work between startTimestamp (exclusive), endTimestamp (inclusive).
    // Business logic implementation goes here.

    result := CronResult{LastSyncTimestamp: endTimestamp}
    return result, nil
}
```

请注意，即使其中一个 cron 计划运行失败，此操作也有效。如果下一个计划至少成功完成一次，它将仍然获得最后一个成功的结果。例如，对于日常cron工作流，如果第一天运行成功而第二天运行失败，则第三天运行仍将使用这些 API 从第一天运行中获取结果。
