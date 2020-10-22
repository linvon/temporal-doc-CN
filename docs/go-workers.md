---
id: go-workers
title: Worker Service
---

工作者或*工作者服务*是承载工作流程和活动实现的服务。工作者轮询 *Temporal 服务*中的任务，执行这些任务，并将任务执行结果传达回 *Temporal 服务*。Temporal 服务由 Temporal 客户端开发，部署和运营。

您可以在新服务或现有服务中运行 Temporal 工作者。使用框架 API 启动 Temporal 工作者，并链接您需要服务执行的所有活动和工作流实现。

```go
package main

import (
	"os"
	"os/signal"

	"github.com/uber-go/tally"
	"go.uber.org/zap"

	"go.temporal.io/sdk/client"
	"go.temporal.io/sdk/worker"
	"go.temporal.io/sdk/workflow"
)

var (
	Taskqueue  = "samples_tq"
)

func main() {
	logger, err := zap.NewDevelopment()
	if err != nil {
		panic(err)
	}

	// The client and worker are heavyweight objects that should be created once per process.
	serviceClient, err := client.NewClient(client.Options{
		HostPort: client.DefaultHostPort,
		Logger: logger,
	})
	if err != nil {
		logger.Fatal("Unable to create client", zap.Error(err))
	}
	defer serviceClient.Close()

	worker := worker.New(serviceClient, TaskQueue, worker.Options{})

	worker.RegisterWorkflow(MyWorkflow)
	worker.RegisterActivity(MyActivity)

	err = worker.Start()
	if err != nil {
		logger.Fatal("Unable to start worker", zap.Error(err))
	}
}

func MyWorkflow(context workflow.Context) error {
	return nil
}

func MyActivity() error {
	return nil
}
```
