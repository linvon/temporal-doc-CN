---
id: go-hello-world
title: Build a  Temporal  "Hello World!" app from scratch
sidebar_label: Build "Hello World!" app
tags: helloworld, go, sdk, tutorial
---

您正在迈出构建更好的应用程序的下一步！

这是针对初次上手[  Temporal  ](https://docs.  Temporal  .io/docs/overview)并且具有一定的Go基本知识的开发人员的教程。我们建议您预留约20分钟的时间来完成它。通过阅读本教程，您将获取以下内容：

- 了解如何设置 Temporal  Go应用程序项目。
- 更加熟悉核心概念和应用程序结构。
- 从零开始使用[ Temporal  Go SDK](https://github.com/Temporalio/sdk-go)构建并测试一个简单的“ Hello World！”  Temporal  Workflow应用程序。

![](../img/docs/hello.png)

本教程重点介绍从头开始构建应用程序的实战。为了更好地理解*为什么*应该使用 Temporal ，我们建议您阅读本教程[运行一个 Temporal 汇款应用程序](https://docs.Temporal.io/docs/go-run-your-first-app)以了解其特性。

所有教程中的代码都可在此仓库中获得 [hello-world Go template repository](https://github.com/Temporalio/hello-world-project-template-go).

## ![](../img/docs/harbor-crane.png) &nbsp;&nbsp;  Go 项目基本框架

在开始之前，请确保您已阅读本教程的[准备条件](https://docs.  Temporal  .io/docs/go-sdk-tutorial-prerequisites)。

在终端中，创建一个名为“ hello-world-project”或类似名称的新项目目录，并使用 cd 进入。

在新项目目录的根目录中，初始化一个新的 Go 模块：

```
go mod init hello-world-project/app
```

然后，添加 Temporal  Go SDK作为项目依赖项：

```
go get go.temporal.io/sdk@latest
```

## ![](../img/docs/apps.png) &nbsp;&nbsp; "Hello World!" 应用

现在，我们准备构建 Temporal 工作流应用程序。我们的应用程序将包括四个部分：

1. 活动：活动只是包含您的业务逻辑的函数。我们将仅格式化一些文本并将其返回。
2. 工作流：工作流是组织活动函数调用的函数。我们的工作流将编排单个活动函数的调用。
3.  Worker ： Worker 托管“活动”和“工作流”代码，并逐段执行代码。
4. 启动器：要启动工作流，我们必须向 Temporal 服务发送一个信号，告诉它跟踪工作流的状态。我们将编写一个单独的函数来执行此操作。

### 活动

首先，让我们定义我们的活动函数，就像 Go 中的任何其他函数一样。活动旨在处理可能导致错误的不确定代码。但是对于本教程，我们要做的只是获取一个字符串，将其附加到“ Hello”，然后将其返回给工作流。

创建 activity.go 并添加以下代码：

``` go
package app

import (
    "fmt"
)

func ComposeGreeting(name string) (string, error) {
    greeting := fmt.Sprintf("Hello %s!", name)
    return greeting, nil
}
```

### 工作流

接下来是我们的工作流。在工作流函数中，您可以配置和组织活动函数的执行。同样，工作流函数的定义与其他任何 Go 函数一样。我们的工作流只是调用`ComposeGreeting()`并返回结果。

创建workflow.go并添加以下代码：

``` go
package app

import (
    "time"

    "go.temporal.io/sdk/workflow"
)

func GreetingWorkflow(ctx workflow.Context, name string) (string, error) {
    options := workflow.ActivityOptions{
        ScheduleToCloseTimeout: time.Minute,
    }
    ctx = workflow.WithActivityOptions(ctx, options)
    var result string
    err := workflow.ExecuteActivity(ctx, ComposeGreeting, name).Get(ctx, &result)
    if err != nil {
        return "", err
    }
    return result, nil
}
```

### 任务队列

任务队列是 Temporal 服务向 Worker s提供信息的方式。启动工作流时，您告诉服务工作流和/或活动使用哪个任务队列作为信息队列。我们将配置 Worker 侦听工作流和活动使用的相同任务队列。由于“任务队列”名称被多个实体使用，因此让我们创建 shared.go 并在其中定义我们的“任务队列”名称：

``` go
package app

const GreetingTaskQueue = "GREETING_TASK_QUEUE"
```

###  Worker 

我们的 Worker 托管工作流和活动函数并一起执行它们一次。通过从任务队列中提取信息，指示 Worker 执行特定函数。运行代码后，它将结果传递回服务。

创建  Worker /main.go 并添加以下代码：

``` go
package main

import (
    "log"

    "go.temporal.io/sdk/client"
    "go.temporal.io/sdk/worker"

    "hello-world-project-template-go/app"
)

func main() {
    // Create the client object just once per process
    c, err := client.NewClient(client.Options{})
    if err != nil {
        log.Fatalln("unable to create Temporal client", err)
    }
    defer c.Close()
    // This worker hosts both Worker and Activity functions
    w := worker.New(c, app.GreetingTaskQueue, worker.Options{})
    w.RegisterWorkflow(app.GreetingWorkflow)
    w.RegisterActivity(app.ComposeGreeting)
    // Start listening to the Task Queue
    err = w.Run(worker.InterruptCh())
    if err != nil {
        log.Fatalln("unable to start Worker", err)
    }
}
```

### 工作流启动器

可以通过SDK或[CLI](https://docs.  Temporal  .io/docs/tctl)两种方法来通过   Temporal   启动工作流。在本教程中，我们使用SDK来启动工作流，这是在大多数生产环境中启动工作流的方式。这是我们刚刚执行此操作的代码：

创建 start/main.go 并添加以下代码:

``` go
package main

import (
    "context"
    "fmt"
    "log"

    "go.temporal.io/sdk/client"

    "hello-world-project-template-go/app"
)

func main() {
    // Create the client object just once per process
    c, err := client.NewClient(client.Options{})
    if err != nil {
        log.Fatalln("unable to create Temporal client", err)
    }
    defer c.Close()
    options := client.StartWorkflowOptions{
        ID:        "greeting-workflow",
        TaskQueue: app.GreetingTaskQueue,
    }
    name := "World"
    we, err := c.ExecuteWorkflow(context.Background(), options, app.GreetingWorkflow, name)
    if err != nil {
        log.Fatalln("unable to complete Workflow", err)
    }
    var greeting string
    err = we.Get(context.Background(), &greeting)
    if err != nil {
        log.Fatalln("unable to get Workflow result", err)
    }
    printResults(greeting, we.GetID(), we.GetRunID())
}

func printResults(greeting string, workflowID, runID string) {
    fmt.Printf("\nWorkflowID: %s RunID: %s\n", workflowID, runID)
    fmt.Printf("\n%s\n\n", greeting)
}
```

##  ![](../img/docs/check.png) &nbsp;测试应用程序

让我们向应用程序中添加一个简单的单元测试，以确保一切正常。创建 workflow_test.go 并添加以下代码：

``` go
package app

import (
    "testing"

    "github.com/stretchr/testify/mock"
    "github.com/stretchr/testify/require"
    "go.temporal.io/sdk/testsuite"
)

func Test_Workflow(t *testing.T) {
    testSuite := &testsuite.WorkflowTestSuite{}
    env := testSuite.NewTestWorkflowEnvironment()
    // Mock activity implementation
    name := "World!"
    env.OnActivity(ComposeGreeting, mock.Anything, name).Return(nil)
    env.ExecuteWorkflow(GreetingWorkflow, name)
    require.True(t, env.IsWorkflowCompleted())
    require.NoError(t, env.GetWorkflowError())
    var greeting string
    require.NoError(t, env.GetWorkflowResult(&greeting))
    require.Equal(t, "Hello World!", greeting)
}
```

从项目根目录运行此命令以执行单元测试：

```
go test
```

## ![](../img/docs/running.png) &nbsp;运行应用程序

要运行该应用程序，我们需要启动工作流和 Worker 。您可以以任何顺序启动它们，但必须在单独的终端窗口中运行每个命令。

要启动 Worker ，请从项目根目录运行以下命令：

```
go run  Worker /main.go
```

要启动工作流，请从项目根目录运行以下命令：

```
go run start/main.go
```

![](../img/docs/confetti.png)

**恭喜，**您已经成功从头开始构建了 Temporal 应用程序！

## ![](../img/docs/wisdom.png) &nbsp;回顾检查

做得好！您现在知道了如何使用Go SDK来构建 Temporal  Workflow应用程序。让我们进行快速回顾，以确保您记得更重要的部分。

![One](../img/docs/one.png) &nbsp;** Temporal 工作流应用程序的最少四个组成是什么？**

1. 活动函数。
2. 工作流函数。
3. 一个用于承载“活动”和“工作流”代码的 Worker 。
4. 用于启动工作流的前端。

![Two](../img/docs/two.png) &nbsp;** Temporal 服务如何向 Worker 获取信息？**

它将信息添加到任务队列。

![Three](../img/docs/three.png) &nbsp;**对或错？ Temporal 活动和工作流函数只是普通的Go函数？**

对

