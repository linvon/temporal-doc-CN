---
id: go-activity-async-completion
title: Asynchronous Activity Completion
---

在某些情况下，我们不能或不希望在完成函数后立即完成活动。例如，您可能有一个需要用户输入才能完成活动的应用程序。您可以使用轮询机制来实现活动，但是更简单且耗费资源较少的实现是异步完成 Temporal 活动。

实现异步完成的活动分为两部分：

1. 该活动从外部系统提供完成所需的信息，并通知 Temporal 服务正在等待该外部回调。
2. 外部服务调用 Temporal 服务以完成活动。

以下示例演示了第一部分：

```go
// Retrieve the activity information needed to asynchronously complete the activity.
//检索异步完成活动所需的活动信息。
activityInfo := activity.GetInfo(ctx)
taskToken := activityInfo.TaskToken

// Send the taskToken to the external service that will complete the activity.
//将taskToken发送到将完成活动的外部服务。
...

// Return from the activity a function indicating that Temporal should wait for an async completion message.
//从活动中返回一个函数，该函数指示Temporal应该等待异步完成信息。
return "", activity.ErrResultPending
```

以下代码演示了如何成功完成活动：

```go
// Instantiate a Temporal service client.
// The same client can be used to complete or fail any number of activities.
// The client is a heavyweight object that should be created once per process.
//实例化临时服务客户端。
//可以使用同一客户端完成或失败任何数量的活动。
//客户端是一个重量级对象，每个进程应创建一次。
serviceClient, err := client.NewClient(client.Options{})

// Complete the activity.
//完成活动。
client.CompleteActivity(taskToken, result, nil)
```

要使活动失败，您可以执行以下操作：

```go
// Fail the activity.
client.CompleteActivity(taskToken, nil, err)
```

以下是该`CompleteActivity`函数的参数：

* `taskToken`：在活动内部检索`ActivityInfo`结构体的`TaskToken`的字段的二进制值。
* `result`：要为活动记录的返回值。此值的类型必须与活动函数声明的返回值的类型匹配。
* `err`：如果活动因错误而终止，则返回的错误代码。

如果`error`不为null，则忽略该`result`字段的值。

