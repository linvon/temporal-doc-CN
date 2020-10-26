---
id: filter-workflows
title: Filter Workflows
---

Temporal支持使用自定义键值对创建工作流，更新工作流代码中的信息，然后使用类似SQL的查询列出/搜索工作流。例如，您可以使用键`city`和`age`创建工作流，然后使用`city = seattle and age > 22`搜索所有工作流。

另外常规的工作流属性，例如开始时间和工作流类型也可以查询。 例如，[从 CLI 列出工作流](https://docs.temporal.io/docs/tctl/#list-closed-or-open-workflow-executions)或使用 List API（[Go](https://pkg.go.dev/go.temporal.io/sdk/client#Client)）时，可以指定以下查询

```sql
WorkflowType = "main.Workflow" and Status != 0 and (StartTime > "2019-06-07T16:46:34-08:00" or CloseTime > "2019-06-07T16:46:34-08:00" order by StartTime desc)
```

## 备注vs搜索属性

Temporal 提供了两种使用键值对创建工作流的方法：备注和搜索属性。备注只能在工作流开始时提供。另外，备注数据没有索引，因此不可搜索。使用 List API 列出工作流时，备注数据也是能看到的。搜索属性数据会建立索引，因此您可以通过查询这些属性来搜索工作流。但是，搜索属性需要使用Elasticsearch。

备注和搜索属性在 Go 客户端的 [StartWorkflowOptions](https://pkg.go.dev/go.temporal.io/sdk/internal#StartWorkflowOptions) 中可用。

```go
type StartWorkflowOptions struct {
    // ...

    // Memo - Optional non-indexed info that will be shown in list workflow.
    Memo map[string]interface{}

    // SearchAttributes - Optional indexed info that can be used in query of List/Scan/Count workflow APIs (only
    // supported when Temporal server is using Elasticsearch). The key and value type must be registered on Temporal server side.
    // Use GetSearchAttributes API to get valid key and corresponding value type.
    SearchAttributes map[string]interface{}
}
```

在Java客户端中，*WorkflowOptions.Builder* 具有用于[备忘](https://javadoc.io/doc/io.temporal/temporal-sdk/latest/io/temporal/client/WorkflowOptions.Builder.html#setMemo-java.util.Map-)和[搜索属性](https://www.javadoc.io/doc/com.uber.cadence/cadence-client/2.6.0/com/uber/cadence/client/WorkflowOptions.Builder.html#setSearchAttributes-java.util.Map-)的类似方法。

备注和搜索属性之间的一些重要区别：

- 备注可以支持所有数据类型，因为它没有索引。搜索属性仅支持基本数据类型（包括String，Int，Float，Bool，Datetime），因为它由 Elasticsearch 索引。
- 备注不限制键名。搜索属性要求在使用键之前将其列出来，因为 Elasticsearch 对索引键有限制。
- 备注不需要 Temporal 集群依赖 Elasticsearch，而搜索属性仅适用于 Elasticsearch。

## 搜索属性 (Go 客户端用例)

使用 Temporal Go 客户端时，请在 [StartWorkflowOptions中 ](https://pkg.go.dev/go.temporal.io/sdk/internal#StartWorkflowOptions)提供键值对作为 SearchAttributes 。

SearchAttributes是`map[string]interface{}`，需要在其中列出键，以便 Temporal 知道属性键的名称和值的类型。映射中提供的值必须与注册的类型相同。

### 列出搜索属性

首先使用 CLI 查询搜索属性列表:

```bash
$ tctl --namespace samples-namespace cl get-search-attr
+---------------------+------------+
|         KEY         | VALUE TYPE |
+---------------------+------------+
| Status              | INT        |
| CloseTime           | INT        |
| CustomBoolField     | DOUBLE     |
| CustomDatetimeField | DATETIME   |
| CustomNamespace     | KEYWORD    |
| CustomDoubleField   | BOOL       |
| CustomIntField      | INT        |
| CustomKeywordField  | KEYWORD    |
| CustomStringField   | STRING     |
| NamespaceId         | KEYWORD    |
| ExecutionTime       | INT        |
| HistoryLength       | INT        |
| RunId               | KEYWORD    |
| StartTime           | INT        |
| WorkflowId          | KEYWORD    |
| WorkflowType        | KEYWORD    |
+---------------------+------------+
```

使用 admin CLI 添加新的搜索属性:

```bash
tctl --namespace samples-namespace adm cl asa --search_attr_key NewKey --search_attr_type string
```

**_search_attr_type_** 的值可为:

- string
- keyword
- int
- double
- bool
- datetime

#### 关键字与字符串

请注意，**关键字**和**字符串**是从 Elasticsearch 获取的概念。**字符串**中的每个单词都被视为可搜索的关键字。对于 UUID，这可能会出现问题，因为 Elasticsearch 会分别索引 UUID 的每个部分。要将整个字符串视为可搜索关键字，请使用**关键字**类型。

例如，键 RunId 的值为“ 2dd29ab7-2dd8-4668-83e0-89cae261cfb1”

- 作为**关键字，**只能与RunId =“ 2dd29ab7-2dd8-4668-83e0-89cae261cfb1”匹配（或在将来使用[正则表达式](https://github.com/uber/cadence/issues/1137)）
- 因为**字符串**将由RunId =“ 2dd8”进行匹配，这可能会导致不必要的匹配

**注意：**字符串类型不能在 Order By 查询中使用。

有一些预先设定的搜索属性可以方便地进行测试：

- CustomKeywordField
- CustomIntField
- CustomDoubleField
- CustomBoolField
- CustomDatetimeField
- CustomStringField

其类型在其名称中已指明。

### 值类型

以下是搜索属性值类型及其对应的 Golang 类型：

- Keyword = string
- Int = int64
- Double = float64
- Bool = bool
- Datetime = time.Time
- String = string

### 限制

我们建议通过对以下各项实施限制来限制 Elasticsearch 索引的数量：

- 密钥数：每个工作流100
- 值大小：每个值 2kb
- 键和值的总大小：每个工作流 40kb

Temporal 保留键，例如 NamespaceId，WorkflowId 和 RunId。这些只能在 List 查询中使用。该值不可更新。

### Upsert 工作流搜索属性

[UpsertSearchAttributes](https://pkg.go.dev/go.temporal.io/sdk/workflow#UpsertSearchAttributes) 用于在工作流代码中添加或更新搜索属性。

可以在[github.com/temporalio/temporal-go-samples中](https://github.com/temporalio/temporal-go-samples/tree/master/searchattributes)找到用于搜索属性的Go样例。

UpsertSearchAttributes 会将属性合并到工作流中的现有 map。例如以下示例工作流代码：

```go
func MyWorkflow(ctx workflow.Context, input string) error {

    attr1 := map[string]interface{}{
        "CustomIntField": 1,
        "CustomBoolField": true,
    }
    workflow.UpsertSearchAttributes(ctx, attr1)

    attr2 := map[string]interface{}{
        "CustomIntField": 2,
        "CustomKeywordField": "seattle",
    }
    workflow.UpsertSearchAttributes(ctx, attr2)
}
```

在第二次调用UpsertSearchAttributes之后，map 将包含：

```go
map[string]interface{}{
    "CustomIntField": 2,
    "CustomBoolField": true,
    "CustomKeywordField": "seattle",
}
```

当前不支持删除字段。为了获得类似的效果，请将字段设置为前哨值。例如，要删除“ CustomKeywordField”，请将其更新为“ impossibleVal”。然后搜索`CustomKeywordField != ‘impossibleVal’`将匹配CustomKeywordField不等于“ impossibleVal”的工作流，其中**包括**未设置 CustomKeywordField 的工作流。

使用`workflow.GetInfo`来获得当前的搜索属性。

###  ContinueAsNew 和 Cron

当执行 [ContinueAsNew](https://docs.temporal.io/docs/go-continue-as-new/) 或使用 [Cron](https://docs.temporal.io/docs/go-distributed-cron/) 时，默认情况下，搜索属性（和备注）将被带到新的运行中。

##  查询功能

通过[CLI列出工作流](https://docs.temporal.io/docs/tctl/#list-closed-or-open-workflow-executions)时，可以使用类似 SQL 的 where 子句查询工作流，也可以使用列表API（[Go](https://pkg.go.dev/go.temporal.io/sdk/client#Client)，[Java](https://static.javadoc.io/com.uber.cadence/cadence-client/2.6.0/com/uber/cadence/WorkflowService.Iface.html#ListWorkflowExecutions-com.uber.cadence.ListWorkflowExecutionsRequest-)）来查询工作流。

请注意，查询时您只会看到来自一个命名空间的工作流。

### 支持的运算符

- AND, OR, ()
- =, !=, >, >=, <, <=
- IN
- BETWEEN ... AND
- ORDER BY

### 默认属性

这些可以通过使用 CLI get-search-attr 命令或 GetSearchAttributes API 来找到。名称和类型如下：

| KEY                 | VALUE TYPE |
| ------------------- | ---------- |
| Status              | INT        |
| CloseTime           | INT        |
| CustomBoolField     | DOUBLE     |
| CustomDatetimeField | DATETIME   |
| CustomNamespace     | KEYWORD    |
| CustomDoubleField   | BOOL       |
| CustomIntField      | INT        |
| CustomKeywordField  | KEYWORD    |
| CustomStringField   | STRING     |
| NamespaceId         | KEYWORD    |
| ExecutionTime       | INT        |
| HistoryLength       | INT        |
| RunId               | KEYWORD    |
| StartTime           | INT        |
| WorkflowId          | KEYWORD    |
| WorkflowType        | KEYWORD    |

这些属性有一些特殊的注意事项：

- Status, CloseTime, NamespaceId, ExecutionTime, HistoryLength, RunId, StartTime, WorkflowId, WorkflowType 是由 Temporal 保留的，并且是只读的
- Status 是 int 到状态的映射:
  - 0 = unknown
  - 1 = running
  - 2 = completed
  - 3 = failed
  - 4 = canceled
  - 5 = terminated
  - 6 = continuedasnew
  - 7 = timedout
- StartTime, CloseTime 和 ExecutionTime 存储为INT, 但支持使用EpochTime（以纳秒为单位）和RFC3339格式的字符串的查询（例如“ 2006-01-02T15：04：05 + 07：00”）
- CloseTime, HistoryLength 仅存在于关闭的工作流中
- ExecutionTime 供 Retry / Cron 用户查询将来将要运行的工作流

要仅列出打开的工作流，请添加`CloseTime = missing`到查询末尾。

如果使用重试或 cron 功能查询将在特定时间范围内开始执行的工作流，则可以在 ExecutionTime 上添加限制。例如：`ExecutionTime > 2019-01-01T10:00:00-07:00`。请注意，如果包含有 ExecutionTime 的限制，则仅返回 cron 或需要重试的工作流。

If you use retry or the cron feature to query workflows that will start execution in a certain time range, you can add predicates on ExecutionTime. For example: `ExecutionTime > 2019-01-01T10:00:00-07:00`. Note that if predicates on ExecutionTime are included, only cron or a workflow that needs to retry will be returned.

### 查询的注意事项

- Pagesize 的默认值为 1000，并且不能大于 10k
- Temporal 时间戳的范围查询（StartTime，CloseTime，ExecutionTime）不能大于9223372036854754775807（maxInt64-1001）
- 按时间范围查询将具有 1ms 的分辨率
- 查询列名称区分大小写
- 检索大量工作流（10M +）时，ListWorkflow 可能需要更长的时间
- 要检索大量工作流而不关心顺序，请使用 ScanWorkflow API
- 为了有效地计算工作流的数量，请使用 CountWorkflow API

## 工具支持

### CLI

从 Temporal 服务的 0.6.0 版本开始支持搜索属性。您还可以从最新的 [CLI Docker 映像](https://hub.docker.com/r/temporalio/tctl)（在 0.6.4 或更高版本上受支持）使用CLI 。

#### 使用搜索属性启动工作流

```bash
tctl --ns samples-namespace workflow start --tq helloWorldGroup --wt main.Workflow --et 60 --dt 10 -i '"vancexu"' -search_attr_key 'CustomIntField | CustomKeywordField | CustomStringField |  CustomBoolField | CustomDatetimeField' -search_attr_value '5 | keyword1 | vancexu test | true | 2019-06-07T16:16:36-08:00'
```

#### 使用 List API 搜索工作流

```bash
tctl --ns samples-namespace wf list -q '(CustomKeywordField = "keyword1" and CustomIntField >= 5) or CustomKeywordField = "keyword2"' -psa
```

```bash
tctl --ns samples-namespace wf list -q 'CustomKeywordField in ("keyword2", "keyword1") and CustomIntField >= 5 and CloseTime between "2018-06-07T16:16:36-08:00" and "2019-06-07T16:46:34-08:00" order by CustomDatetimeField desc' -psa
```

要仅列出打开的工作流，请添加`CloseTime = missing`到查询末尾。

请注意，查询可以支持多种类型的过滤器：

```bash
tctl --ns samples-namespace wf list -q 'WorkflowType = "main.Workflow" and (WorkflowId = "1645a588-4772-4dab-b276-5f9db108b3a8" or RunId = "be66519b-5f09-40cd-b2e8-20e4106244dc")'
```

```bash
tctl --ns samples-namespace wf list -q 'WorkflowType = "main.Workflow" StartTime > "2019-06-07T16:46:34-08:00" and CloseTime = missing'
```

### Web UI 支持

从0.20.0版本开始，[Temporal Web](https://github.com/temporalio/temporal-web) 中支持查询。使用“基本/高级”按钮切换到“高级”模式，然后在搜索框中键入查询。

## 本地测试

1. 将 Docker 内存增加到 6GB 以上。进入到Docker-> Preferences-> Advanced-> Memory
2. 获取临时 Docker 撰写文件。运行`curl -L https://github.com/temporalio/temporal/releases/latest/download/docker.tar.gz | tar -xz --strip-components 1 docker/docker-compose-es.yml`
3. 使用以下命令启动 Temporal Docker（其中包含 Apache Kafka，Apache Zookeeper 和 Elasticsearch） `docker-compose -f docker-compose-es.yml up`
4. 从 Docker 输出日志中，确保 Elasticsearch 和 Temporal 正确启动。如果遇到磁盘空间不足错误，请尝试`docker system prune -a --volumes`
5. 注册一个本地命名空间并开始使用它。 `tctl --ns samples-namespace n re`

注意：启动具有搜索属性但没有 Elasticsearch 的工作流将照常成功，但是将不可搜索，也不会显示在列表结果中。

