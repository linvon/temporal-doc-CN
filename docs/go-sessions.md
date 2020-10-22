---
id: go-sessions
title: Sessions
---

会话框架提供了一个简单的交互，用于在单个工作线程上调度多个活动，而无需您手动指定任务队列名称。它还包括诸如**并发会话限制**和**工作程序故障检测之类的功能**。

## 使用案例

- **文件处理**：您可能想要实现一个工作流程，可以下载文件，对其进行处理，然后上传修改后的版本。如果将这三个步骤实现为三个不同的活动，则所有这些步骤均应由同一工作人员执行。
- **机器学习模型训练**：训练机器学习模型通常包括三个阶段：下载数据集，优化模型以及上载训练后的参数。由于模型可能消耗大量资源（例如GPU内存），因此需要限制在主机上处理的模型数量。

## 基本用法

在使用会话框架编写工作流代码之前，您需要配置工作程序以处理会话。要做到这一点，设置`worker.Options`的`EnableSessionWorker`字段为`true`来启动你的工作者。

会话框架提供的最重要的API是`workflow.CreateSession()`和`workflow.CompleteSession()`。基本思想是，会话中执行的所有活动将由同一工作人员处理，这两个API允许您创建新会话，并在所有活动完成后关闭它们。

这是这两个API的详细说明：

```go
type SessionOptions struct {
  // ExecutionTimeout: required, no default.
  //     Specifies the maximum amount of time the session can run.
  // ExecutionTimeout：必需，无默认值。
  //指定会话可以运行的最长时间。
  ExecutionTimeout time.Duration

  // CreationTimeout: required, no default.
  //     Specifies how long session creation can take before returning an error.
  // CreationTimeout：必需，无默认值。
  //指定创建会话过程的超时时间。
  CreationTimeout  time.Duration
}

func CreateSession(ctx Context, sessionOptions *SessionOptions) (Context, error)
```

`CreateSession()`接受`workflow.Context`，`sessionOptions`并返回一个新上下文，其中包含创建的会话的元数据信息（以下称为**会话上下文**）。调用它时，它将检查在`ActivityOptions`中指定的任务队列名称（如果没有在`ActivityOptions`中指定，则检查`StartWorkflowOptions`），并在轮询该任务队列的工作线程之一上创建会话。

返回的会话上下文应用于执行属于该会话的所有活动。如果执行此会话的工作程序死掉或`CompleteSession()`被调用，则上下文将被取消。使用返回的会话上下文执行活动时，如果会话框架检测到执行此会话的工作程序已死亡，则会返回`workflow.ErrSessionFailed`错误。活动失败不会影响会话状态，因此您仍然需要处理活动返回的错误，并在必要时调用`CompleteSession()`。

如果传入的上下文已经包含打开的会话，则`CreateSession()`将返回错误。如果当前所有工作人员都很忙并且无法处理新的会话，则框架将继续重试，直到到达您在`SessionOptions`对象指定的`CreationTimeout`超时为止，然后返回错误（有关更多详细信息，请参阅“ **并发会话限制”**部分）。

```go
func CompleteSession(ctx Context)
```

`CompleteSession()`释放保留在工作者上的资源，因此在不再需要会话时立即调用它很重要。它将取消会话上下文，并因此取消使用该会话上下文的所有活动。请注意，`CompleteSession()`在失败的会话上调用是安全的，这意味着您可以在成功创建会话之后从`defer`函数中调用它。

### 示例代码

```go
func FileProcessingWorkflow(ctx workflow.Context, fileID string) (err error) {
  ao := workflow.ActivityOptions{
    ScheduleToStartTimeout: time.Second * 5,
    StartToCloseTimeout:    time.Minute,
  }
  ctx = workflow.WithActivityOptions(ctx, ao)

  so := &workflow.SessionOptions{
    CreationTimeout:  time.Minute,
    ExecutionTimeout: time.Minute,
  }
  sessionCtx, err := workflow.CreateSession(ctx, so)
  if err != nil {
    return err
  }
  defer workflow.CompleteSession(sessionCtx)

  var fInfo *fileInfo
  err = workflow.ExecuteActivity(sessionCtx, downloadFileActivityName, fileID).Get(sessionCtx, &fInfo)
  if err != nil {
    return err
  }

  var fInfoProcessed *fileInfo
  err = workflow.ExecuteActivity(sessionCtx, processFileActivityName, *fInfo).Get(sessionCtx, &fInfoProcessed)
  if err != nil {
    return err
  }

  return workflow.ExecuteActivity(sessionCtx, uploadFileActivityName, *fInfoProcessed).Get(sessionCtx, nil)
}
```

## 会话元数据

```go
type SessionInfo struct {
  // A unique Id for the session
  SessionID         string

  // The hostname of the worker that is executing the session
  HostName          string

  // ... other unexported fields
}

func GetSessionInfo(ctx Context) *SessionInfo
```

会话上下文还存储一些会话元数据，可以由`GetSessionInfo()`API 检索。如果传入的上下文不包含任何会话元数据，则此API将返回一个`nil`指针。

## 并发会话限制

要限制在工作程序上运行的并发会话数，请将`worker.Options`的`MaxConcurrentSessionExecutionSize`字段设置为所需的值。默认情况下，此字段设置为非常大的值，因此如果不需要限制，则无需手动设置它。

如果工作人员达到此限制，则`CreateSession()`在完成任何现有会话之前，它将不接受任何新请求。如果无法在`CreationTimeout`内创建会话，则`CreateSession()`会返回错误。

## 重新创建会话

对于长时间运行的会话，当所有活动需要由同一工作人员执行时，您可能希望使用`ContinueAsNew`功能将工作流程分为多个运行。该`RecreateSession()`API专为此类用例而设计。

```go
func RecreateSession(ctx Context, recreateToken []byte, sessionOptions *SessionOptions) (Context, error)
```

它的用法大致与`CreateSession()`相同，除了它还接受`recreateToken`参数，这是在与上一个会话相同的工作线程上创建新会话所需的。您可以通过调用`SessionInfo`对象的`GetRecreateToken()`方法来获取令牌。

```go
token := workflow.GetSessionInfo(sessionCtx).GetRecreateToken()
```

## Q & A

### 有完整的例子吗？

是的，temporal-go-samples 存储库中的[文件处理示例](https://github.com/temporalio/temporal-go-samples/blob/master/cmd/samples/fileprocessing/workflow.go)已更新为使用会话框架。

Yes, the [file processing example](https://github.com/temporalio/temporal-go-samples/blob/master/cmd/samples/fileprocessing/workflow.go) in the temporal-go-samples repo has been updated to use the session framework.

### 如果工作者死亡，我的活动将会怎样？

如果您的活动已经安排好了，它将被取消。否则，`workflow.ExecuteActivity()`调用会出现`workflow.ErrSessionFailed`错误。

### 并发会话限制是每个工作者的还是每个主机的？

是每个工作者的，因此，如果您打算使用该功能，请确保主机上仅运行一个工作者。

## 未来的工作

-

现在，如果工作进程终止，则会话被视为失败。但是，对于某些用例，您可能只在乎工作主机是否还活着。对于这些用例，如果重新启动工作进程，则应自动重新建立会话。

-

当前的实现假设所有会话都消耗相同类型的资源，并且只有一个全局限制。我们的计划是允许您指定会话将消耗的资源类型，并对不同类型的资源施加不同的限制。
