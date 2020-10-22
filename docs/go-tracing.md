---
id: go-tracing
title: Tracing and Context Propagation
---

## 追踪

Go客户端通过[OpenTracing](https://opentracing.io/) 提供分布式跟踪支持。可以通过在客户端和工作者实例化期间分别在 [ClientOptions](https://pkg.go.dev/go.temporal.io/sdk/internal#ClientOptions) 和 [WorkerOptions](https://pkg.go.dev/go.temporal.io/sdk/internal#WorkerOptions) 中提供 [opentracing.Tracer](https://pkg.go.dev/github.com/opentracing/opentracing-go#Tracer) 实现来配置跟踪。跟踪使您可以查看工作流的调用图及其活动，子工作流等。有关如何配置和利用跟踪的更多详细信息，请参阅 [OpenTracing文档](https://opentracing.io/docs/getting-started/)。OpenTracing支持使用[Jaeger](https://www.jaegertracing.io/)已得到了验证，但是[这里](https://opentracing.io/docs/supported-tracers/)提到的其他实现也应该起作用。跟踪利用客户端提供的通用 context 传输来支持。

## Context 传输

我们提供了一种在工作流中传输自定义 context 的标准方法。 [ClientOptions](https://pkg.go.dev/go.temporal.io/sdk/internal#ClientOptions) 和 [WorkerOptions](https://pkg.go.dev/go.temporal.io/sdk/internal#WorkerOptions) 允许配置 context 传输器。 context 传输器提取并传递整个工作流程中`context.Context` 和`workflow.Context`对象中存在的信息。配置了 context 传输器后，您就能够像通常在Go中一样访问 context 对象中的所需值。作为示例，Go客户端实现了一个[追踪 context 传输器](https://github.com/temporalio/temporal-go-sdk/blob/master/internal/tracer.go)。

### 服务器端头部支持

在服务器端，Temporal 提供了一种在不同的工作流转换之间传输其头部的机制。

```proto
message Header {
    map<string, bytes> fields = 1;
}
```

客户端利用它来传递选定的 context 信息。[HeaderReader](https://pkg.go.dev/go.temporal.io/sdk/internal#HeaderReader) 和[HeaderWriter](https://pkg.go.dev/go.temporal.io/sdk/internal#HeaderWriter)是允许读取和写入 Temporal 服务器头部的接口。客户端已经提供了这些[实现](https://github.com/temporalio/temporal-go-sdk/blob/master/internal/headers.go)。`HeaderWriter`在标题中设置一个字段。头部是一个映射，因此多次为同一键设置一个值将覆盖以前的值。`HeaderReader`遍历头部 map，并在每个键/值对上运行提供的处理程序函数，从而使您可以处理感兴趣的字段。

```go
type HeaderWriter interface {
  Set(string, []byte)
}

type HeaderReader interface {
  ForEachKey(handler func(string, []byte) error) error
}
```

### Context 传输器

 context 传输器需要实现以下四种方法，以在工作流中传输所选的 context ：

- `Inject`旨在从Go [context.Context](https://golang.org/pkg/context/#Context)对象中选择感兴趣的 context 键，然后使用 [HeaderWriter](https://pkg.go.dev/go.temporal.io/sdk/internal#HeaderWriter) 接口将其写入头部
- `InjectFromWorkflow`与上述相同，但操作的是 [workflow.Context ](https://pkg.go.dev/go.temporal.io/sdk/internal#Context)对象
- `Extract`读取头部并将感兴趣的信息放回 [context.Context](https://golang.org/pkg/context/#Context) 对象
- `ExtractToWorkflow`与上述相同，但操作的是[workflow.Context](https://pkg.go.dev/go.temporal.io/sdk/internal#Context) 对象

[tracing context propagator](https://github.com/temporalio/temporal-go-sdk/blob/master/internal/tracer.go)
展示了实现 context 传输的示例。

```go
type ContextPropagator interface {
  Inject(context.Context, HeaderWriter) error

  Extract(context.Context, HeaderReader) (context.Context, error)

  InjectFromWorkflow(Context, HeaderWriter) error

  ExtractToWorkflow(Context, HeaderReader) (Context, error)
}
```

## Q & A

### 有完整的例子吗？

[context propagation sample](https://github.com/temporalio/temporal-go-samples/blob/master/ctxpropagation/workflow.go)
这里的样例展示了如何配置自定义键的 context 传输器，并展示 context 是如何在工作流和活动的定义传输的。

### 我可以配置多个 context 传输器吗？

是的，我们建议您配置多个 context 传输器，每个传输器用于传输特定类型的 context 。
