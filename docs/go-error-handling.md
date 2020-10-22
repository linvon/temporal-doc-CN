---
id: go-error-handling
title: Error Handling
---

活动或子工作流程可能会失败，并且您可以根据不同的错误情况以不同的方式处理错误。

如果活动返回`errors.New()`或`fmt.Errorf()`错误，则这些错误将转换为`*temporal.ApplicationError`并封装在`*temporal.ActivityTaskError`或`*temporal.ChildWorkflowExecutionError`中。

如果活动返回的错误为 `temporal.NewRetryableApplicationError("error message", details)`，则该错误将返回为`*temporal.ApplicationError`。

还有其他类型的错误如`*temporal.TimeoutError`，`*temporal.CanceledError`和 `*temporal.PanicError`。以下是错误处理代码的示例：

```go
err := workflow.ExecuteActivity(ctx, MyActivity, ...).Get(ctx, nil)
if err != nil {
	var applicationErr *ApplicationError
	if errors.As(err, &applicationErr) {
		// retrieve error message
		fmt.Println(applicationError.Error())

		// handle activity errors (created via NewApplicationError() API)
		var detailMsg string // assuming activity return error by NewApplicationError("message", true, "string details")
		applicationErr.Details(&detailMsg) // extract strong typed details

		// handle activity errors (errors created other than using NewApplicationError() API)
		switch err.OriginalType() {
		case "CustomErrTypeA":
			// handle CustomErrTypeA
		case CustomErrTypeB:
			// handle CustomErrTypeB
		default:
			// newer version of activity could return new errors that workflow was not aware of.
		}
	}

	var canceledErr *CanceledError
	if errors.As(err, &canceledErr) {
		// handle cancellation
	}

	var timeoutErr *TimeoutError
	if errors.As(err, &timeoutErr) {
		// handle timeout, could check timeout type by timeoutErr.TimeoutType()
        switch err.TimeoutType() {
        case commonpb.ScheduleToStart:
                // Handle ScheduleToStart timeout.
        case commonpb.StartToClose:
                // Handle StartToClose timeout.
        case commonpb.Heartbeat:
                // Handle heartbeat timeout.
        default:
        }
	}

	var panicErr *PanicError
	if errors.As(err, &panicErr) {
		// handle panic, message and stack trace are available by panicErr.Error() and panicErr.StackTrace()
	}
}
```
