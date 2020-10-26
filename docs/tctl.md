---
id: tctl
title: tctl (CLI)
sidebar_label: tctl (CLI)
---

Temporal CLI是一个命令行工具，可用于在 Temporal 服务上执行各种任务。它可以执行命名空间操作（例如注册、更新和描述）以及工作流操作（例如开始工作流、显示工作流历史记录和信号通知工作流）。

## 使用 CLI

可以从 Docker Hub 镜像 _temporalio/tctl_ 直接使用 Temporal CLI，也可以在本地编译CLI工具。

使用 docker 映像描述命名空间的示例：

```
docker run --rm temporalio/tctl:0.29.0 --namespace samples-namespace namespace describe
```

在Docker 18.03及更高版本上，您可能会收到“connection refused”错误。您可以通过将主机设置为“ host.docker.internal”来解决此问题（有关更多信息，请参见[此处](https://docs.docker.com/docker-for-mac/networking/#use-cases-and-workarounds)）。

```
docker run --network=host --rm temporalio/tctl:0.29.0 --namespace samples-namespace namespace describe
```

要在本地构建CLI工具，请克隆[Temporal服务存储库](https://github.com/temporalio/temporal)并运行 `make bins`。这将产生一个名为的可执行文件`tctl`。在本地构建中，用于描述命名空间的相同命令如下所示：

```
./tctl --namespace samples-namespace namespace describe
```

为了简洁起见，下面的示例命令将使用 `./tctl` 。

## 环境变量

为重复的参数设置环境变量可以缩短 CLI 命令。

- **TEMPORAL_CLI_ADDRESS** - Temporal 前端服务的主机和端口 (默认: `127.0.0.1:7233`)
- **TEMPORAL_CLI_NAMESPACE** - 工作流命名空间, 这样你就不用指定 `--namespace` (默认命名空间: `default`)
- **TEMPORAL_CLI_TLS_CA** - 服务器证书颁发机构证书文件的路径
- **TEMPORAL_CLI_TLS_CERT** - 公共x509证书文件的路径，用于相互 TLS 身份验证
- **TEMPORAL_CLI_TLS_KEY** - 相互TLS身份验证的私钥文件的路径

## Quick Start

- 运行 `./tctl -h` 获取有关顶级命令和全局选项的帮助
- 运行 `./tctl namespace -h` 获取有关命名空间操作的帮助
- 运行 `./tctl workflow -h` 获取有关工作流操作的帮助
- 运行 `./tctl taskqueue -h` 获取有关任务队列操作的帮助

**注意：**使用 CLI 之前，请确保您已运行 Temporal 服务器

### 命名空间操作的例子

- 注册一个名为“ samples-namespace”的新命名空间：

```
./tctl --namespace samples-namespace namespace register
# OR using short alias
./tctl --ns samples-namespace n re
```

- 查看 "samples-namespace" 详细信息:

```
./tctl --namespace samples-namespace namespace describe
```

### 工作流操作的例子

以下示例假定设置了`TEMPORAL_CLI_NAMESPACE`环境变量。

#### 运行工作流

启动工作流并查看其进度。在工作流完成之前，该命令不会完成。

```
./tctl workflow run --tq hello-world --wt Workflow --et 60 -i '"temporal"'

# view help messages for workflow run
./tctl workflow run -h
```

简要说明：要运行工作流，用户必须指定以下内容：

1. 任务队列名称 (--tq)
2. 工作流类型 (--wt)
3. 以秒为单位的从执行开始到关闭的超时 (--et)
4. 以 JSON 格式输入 (--i) (可选)

上面的示例使用[此示例工作流](https://github.com/temporalio/go-samples/blob/master/helloworld/helloworld.go) ，并使用字符串作为`-i '"temporal"'`参数的输入。单引号（`''`）用于将输入包装为 JSON。

**注意：**您需要启动 Worker ，以便工作流可以运行。（在 temporal-go-samples 中运行`make && ./bin/helloworld -m worker`以启动 Worker ）

#### 显示任务队列中正在运行的 Worker 

```
./tctl taskqueue desc --tq hello-world
```

#### 启动工作流

```
./tctl workflow start --tq hello-world --wt Workflow --et 60 -i '"temporal"'

# view help messages for workflow start
./tctl workflow start -h

# for a workflow with multiple inputs, provide a separate -i flag for each of them
./tctl workflow start --tq hello-world --wt WorkflowWith3Args --et 60 -i '"your_input_string"' -i 'null' -i '{"Name":"my-string", "Age":12345}'
```

工作流`start`命令与该`run`命令相似，但是在启动工作流后立即返回工作流 ID 和 run_id。使用`show`命令查看工作流的历史记录/进度。

##### 启动/运行工作流时重复使用相同的工作流 ID

使用选项`--workflowidreusepolicy`或`--wrp`配置工作流 ID 重用策略。

**选项0 AllowDuplicateFailedOnly：**当具有相同工作流 ID 的工作流尚未运行且上次执行关闭状态为*[终止，取消，超时，失败]中的*一种时，允许使用相同的工作流 ID 启动工作流执行。

**选项1 AllowDuplicate：**当具有相同工作流 ID 的工作流尚未运行时，允许使用相同的工作流 ID 启动工作流执行。

**选项2 RejectDuplicate：**不允许使用与以前的工作流相同的工作流 ID 开始执行工作流。

```
# use AllowDuplicateFailedOnly option to start a Workflow
./tctl workflow start --tq hello-world --wt Workflow --et 60 -i '"temporal"' --wid "<duplicated workflow id>" --wrp AllowDuplicateFailedOnly

# use AllowDuplicate option to run a workflow
./tctl workflow run --tq hello-world --wt Workflow --et 60 -i '"temporal"' --wid "<duplicated workflow id>" --wrp AllowDuplicate
```

##### 启动带有备忘录的工作流

备注是不可变的键/值对，可以在启动工作流时将其附加到工作流中。列出工作流时这些也是可见的。有关备忘的更多信息，请参见 [此处](https://docs.temporal.io/docs/filter-workflows/#memo-vs-search-attributes)。

```
tctl wf start -tq hello-world -wt Workflow -et 60 -i '"temporal"' -memo_key ‘“Service” “Env” “Instance”’ -memo ‘“serverName1” “test” 5’
```

#### 显示工作流历史记录

```
./tctl workflow show -w 3ea6b242-b23c-4279-bb13-f215661b4717 -r 866ae14c-88cf-4f1e-980f-571e031d71b0
# a shortcut of this is (without -w -r flag)
./tctl workflow showid 3ea6b242-b23c-4279-bb13-f215661b4717 866ae14c-88cf-4f1e-980f-571e031d71b0

# if run_id is not provided, it will show the latest run history of that workflow_id
./tctl workflow show -w 3ea6b242-b23c-4279-bb13-f215661b4717
# a shortcut of this is
./tctl workflow showid 3ea6b242-b23c-4279-bb13-f215661b4717
```

#### 显示工作流执行信息

```
./tctl workflow describe -w 3ea6b242-b23c-4279-bb13-f215661b4717 -r 866ae14c-88cf-4f1e-980f-571e031d71b0
# a shortcut of this is (without -w -r flag)
./tctl workflow describeid 3ea6b242-b23c-4279-bb13-f215661b4717 866ae14c-88cf-4f1e-980f-571e031d71b0

# if run_id is not provided, it will show the latest workflow execution of that workflow_id
./tctl workflow describe -w 3ea6b242-b23c-4279-bb13-f215661b4717
# a shortcut of this is
./tctl workflow describeid 3ea6b242-b23c-4279-bb13-f215661b4717
```

#### 列出关闭或打开的工作流执行

```
./tctl workflow list

# default will only show one page, to view more items, use --more flag
./tctl workflow list -m
```

使用**--query**可以使用类似SQL查询的方式列出工作流：

```
./tctl workflow list --query "WorkflowType='main.SampleParentWorkflow' AND CloseTime = missing "
```

这将返回所有打开的工作流，其工作流类型为 "main.SampleParentWorkflow"。

#### 查询工作流执行

```
# use custom query type
./tctl workflow query -w <wid> -r <rid> --qt <query-type>

# use build-in query type "__stack_trace" which is supported by Temporal SDK
./tctl workflow query -w <wid> -r <rid> --qt __stack_trace
# a shortcut to query using __stack_trace is (without --qt flag)
./tctl workflow stack -w <wid> -r <rid>
```

#### 信号、取消、终止工作流

```
# signal
./tctl workflow signal -w <wid> -r <rid> -n <signal-name> -i '"signal-value"'

# cancel
./tctl workflow cancel -w <wid> -r <rid>

# terminate
./tctl workflow terminate -w <wid> -r <rid> --reason
```

终止正在运行的工作流执行将在历史记录中将 WorkflowExecutionTerminated 事件记录为关闭事件。终止执行的工作流不再调度决策任务。取消正在运行的工作流执行，将在历史记录中记录 WorkflowExecutionCancelRequested 事件，并调度新的决策任务。取消后，工作流有机会进行一些清理工作。

#### 批量信号、取消、终止工作流

批处理作业基于列出工作流查询（**--query**）。它支持信号、取消和终止作为批处理作业类型。对于终止工作流的批处理作业，它将递归终止子工作流。

启动批处理作业（使用信号作为批处理类型）:

```
tctl --ns samples-namespace batch start --query "WorkflowType='main.SampleParentWorkflow' AND CloseTime=missing" --reason "test" --bt signal --sig testname
This batch job will be operating on 5 workflows.
Please confirm[Yes/No]:yes
{
  "jobId": "<batch-job-id>",
  "msg": "batch job is started"
}

```

您需要记住 JobId 或使用 List 命令来获取所有批处理作业：

```
tctl --ns samples-namespace batch list
```

描述批处理作业的进度：

```
tctl --ns samples-namespace batch desc -jid <batch-job-id>
```

终止批处理作业：

```
tctl --ns samples-namespace batch terminate -jid <batch-job-id>
```

注意，批处理执行的操作不会通过终止该批处理而回滚。但是，您可以使用 reset 来回滚您的工作流。

#### 重新启动、重置工作流

使用“重置”命令可以将工作流重置到特定点并从那里继续运行。

有很多使用场景：

- 从头开始使用相同的启动参数重新运行失败的工作流。
- 从故障点重新运行失败的工作流，而不会丢失已实现的进度（历史）。
- 部署新代码后，重置打开的工作流以使工作流运行到不同的流程。

您可以重置为一些预定义的事件类型：

```
./tctl workflow reset -w <wid> -r <rid> --reset_type <reset_type> --reason "some_reason"
```

- FirstDecisionCompleted: 重置到历史记录的开头。
- LastDecisionCompleted: 重置到历史记录的末尾。
- LastContinuedAsNew: 重置到上一次运行的历史记录的末尾。

如果您熟悉 Temporal 历史事件，则还可以使用以下方法将其重置为任何决策完成事件：

```
./tctl workflow reset -w <wid> -r <rid> --event_id <decision_finish_event_id> --reason "some_reason"
```

注意事项：

- 重置后，将使用相同的工作流编号开始新的运行。但是如果工作流（workflowId）正在运行，则当前运行将终止。
- Decision_finish_event_id 是以下类型的事件的ID：DecisionTaskComplete / DecisionTaskFailed / DecisionTaskTimeout。
- 要从头开始重新启动工作流，请重置为第一个决策任务完成事件。

要重置多个工作流，可以使用批重置命令：

```
./tctl workflow reset-batch --input_file <file_of_workflows_to_reset> --reset_type <reset_type> --reason "some_reason"
```

#### 从不良部署中恢复-自动重置工作流

如果不良部署使工作流进入错误状态，则您可能需要将工作流重置为不良部署开始运行的时刻。但是通常很难找出所有受影响的工作流以及每个工作流的每个重置点。在这种情况下，自动重置将在给定错误的部署标识符的情况下自动重置所有工作流。

让我们熟悉一些概念。每个部署都有一个标识符，我们称其为“ **Binary Checksum** ”，因为它通常是由二进制文件的md5sum生成的。对于一个工作流，每个二进制的校验和将关联一个**自动复位点**，其中包含一个 **runid**，一个 **eventID **和二进制/部署生成工作流中的第一个决定的**创建时间**。

为了找出要重置不良部署的**二进制校验和**，您应该知道至少有一个工作流处于不良状态。使用带**--reset_points_only **选项的 describe 命令可显示所有重置点：

```
./tctl wf desc -w <WorkflowId>  --reset_points_only
+----------------------------------+--------------------------------+--------------------------------------+---------+
|         BINARY CHECKSUM          |          CREATE TIME           |                RUNID                 | EVENTID |
+----------------------------------+--------------------------------+--------------------------------------+---------+
| c84c5afa552613a83294793f4e664a7f | 2019-05-24 10:01:00.398455019  | 2dd29ab7-2dd8-4668-83e0-89cae261cfb1 |       4 |
| aae748fdc557a3f873adbe1dd066713f | 2019-05-24 11:01:00.067691445  | d42d21b8-2adb-4313-b069-3837d44d6ce6 |       4 |
...
...
```

然后使用此命令告诉 Temporal 自动重置受不良部署影响的所有工作流。该命令会将错误的二进制校验和存储到命名空间信息中，并触发重置所有工作流的进程。

```
./tctl --ns <YourNamespace> namespace update --add_bad_binary aae748fdc557a3f873adbe1dd066713f  --reason "rollback bad deployment"
```

在将错误的二进制校验和添加到命名空间时，Temporal 将不会将任何决策任务分派到错误的二进制文件。因此，请确保您已回滚到良好的部署（或使用错误修复来推出新的组件）。否则，自动重置后您的工作流将无法取得任何进展。

#### 安全连接到 Temporal 集群

`tctl` 支持可选的Transport Level Security（TLS），用于与 Temporal 安全通信，服务器身份验证和客户端身份验证（相互TLS）。

`--tls_ca_path=<certificate file path>`命令行参数为`tctl`正在连接的验证服务器传递证书颁发机构（CA）证书。

`--tls_cert_path=<certificate file path>`命令行参数为服务器传递证书以验证客户端（`tctl`）身份。同时`--tls_key_path`也需提供。

`--tls_key_path=<private key file path>`传递用于与服务器安全通信的私钥的命令行参数。同时`--tls_key_path`也需提供。

TLS命令行参数可以通过各自的环境变量提供，以缩短命令行。