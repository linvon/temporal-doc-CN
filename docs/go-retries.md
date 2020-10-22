---
id: go-retries
title: Activity and Workflow Retries
---

由于各种中间条件，活动和工作流可能会失败。在这些情况下，我们想重试失败的活动或子工作流，甚至父工作流。这可以通过提供可选的重试策略来实现。重试策略如下所示：

``` go
// RetryPolicy defines the retry policy.
RetryPolicy struct {
 		//第一次重试的退避间隔。如果系数为1.0，则会被用于每次重试。
    //必填，无默认值。
    // Backoff interval for the first retry. If coefficient is 1.0 then it is used for all retries.
    // Required, no default value.
    InitialInterval time.Duration

  	//用于计算下一个重试退避间隔的系数。
    //下一个重试间隔是前一个间隔乘以该系数。
    //必须为1或更大。默认值为2.0。
    // Coefficient used to calculate the next retry backoff interval.
    // The next retry interval is previous interval multiplied by this coefficient.
    // Must be 1 or larger. Default is 2.0.
    BackoffCoefficient float64

  	//重试之间的最大退避间隔。间隔以指数回退增加。
    //此值是间隔的上限。默认值为初始间隔的100倍。
    // Maximum backoff interval between retries. Exponential backoff leads to interval increase.
    // This value is the cap of the interval. Default is 100x of initial interval.
    MaximumInterval time.Duration

  	//最大尝试次数。如果超过该时间，重试将停止，即使尚未过期也是如此。
    //如果未设置或设置为0，则表示无限制
    // Maximum number of attempts. When exceeded the retries stop even if not expired yet.
    // If not set or set to 0, it means unlimited
    MaximumAttempts int32

    //不可恢复的错误。这是可选的。如果错误类型与此列表匹配，则 Temporal 服务器将停止重试。
    // 注意：
    // - 取消不是失败，因此不会被重试，
    // - 仅可重试 StartToClose 或心跳超时。
    // Non-Retriable errors. This is optional. Temporal server will stop retry if error type matches this list.
    // Note:
    //  - cancellation is not a failure, so it won't be retried,
    //  - only StartToClose or Heartbeat timeouts are retryable.
    NonRetryableErrorTypes []string
}
```

要启用重试，请在执行它们时向`ActivityOptions`或`ChildWorkflowOptions` 提供自定义重试策略。

``` go
expiration := time.Minute * 10
retryPolicy := &temporal.RetryPolicy{
    InitialInterval:    time.Second,
    BackoffCoefficient: 2,
    MaximumInterval:    expiration,
    MaximumAttempts:    5,
}
ao := workflow.ActivityOptions{
    ScheduleToStartTimeout: expiration,
    StartToCloseTimeout:    expiration,
    HeartbeatTimeout:       time.Second * 30,
    RetryPolicy:            retryPolicy, // Enable retry.
}
ctx = workflow.WithActivityOptions(ctx, ao)
activityFuture := workflow.ExecuteActivity(ctx, SampleActivity, params)
```

如果活动在失败之前对其进度进行了心跳报告，则重试尝试将包含该进度，因此活动实现可以从失败的进度中恢复，例如：

``` go
func SampleActivity(ctx context.Context, inputArg InputParams) error {
    startIdx := inputArg.StartIndex
    if activity.HasHeartbeatDetails(ctx) {
        // Recover from finished progress.
        var finishedIndex int
        if err := activity.GetHeartbeatDetails(ctx, &finishedIndex); err == nil {
            startIdx = finishedIndex + 1 // Start from next one.
        }
    }

    // Normal activity logic...
    for i:=startIdx; i<inputArg.EndIdx; i++ {
        // Code for processing item i goes here...
        activity.RecordHeartbeat(ctx, i) // Report progress.
    }
}
```

与活动重试类似，您需要提供重试策略`ChildWorkflowOptions`以启用子工作流的重试。要为父工作流启用重试，请在通过启动工作流时提供重试策略`StartWorkflowOptions`。

`RetryPolicy`使用时，工作流的历史事件会有一些细微的变化。

对于使用`RetryPolicy`的活动：

- `ActivityTaskScheduled`事件会有扩展的`ScheduleToStartTimeout`和`ScheduleToCloseTimeout`项。
- `ActivityTaskStarted`事件将不会在历史上出现，除非活动完成或不再重试失败。这是为了避免事先记录了该`ActivityTaskStarted`事件，但之后又失败并重试。如果活动正在重试，使用`DescribeWorkflowExecution` API将返回`PendingActivityInfo`并包含`attemptCount`。

对于使用`RetryPolicy`的工作流：

- 如果工作流失败并需要重试，则该失败工作流执行将因`ContinueAsNew`事件而关闭。该事件将拥有设置在`RetryPolicy`中的`ContinueAsNewInitiator`和一个为下一次重试尝试设置的新的`RunId`。
- 新的尝试将立即创建。但是第一个决策任务要等到退避时长才被安排，退避时长也将作为`firstDecisionTaskBackoffSeconds`记录在新运行的`WorkflowExecutionStartedEventAttributes`事件中。