---
id: go-side-effect
title: SideEffect
---

`workflow.SideEffect` 对于简短的，不确定的代码段（例如获取随机值或生成UUID）很有用。它只执行提供的函数一次，并将其结果记录到工作流历史记录中。`workflow.SideEffect`不会在重放时重新执行，而是返回记录的结果。可以将其视为“内联”活动。需要注意的一点是，与 Temporal 保证的活动最多执行一次不同，使用`workflow.SideEffect`时没有这种保证。在某些故障情况下，`workflow.SideEffect`最终可能会多次执行函数。

`SideEffect`失败的唯一方法是 panic，这会导致决策任务失败。超时后，Temporal重新安排然后重新执行决策任务，从而给`SideEffect`另一次执行成功的机会。除了通过记录的返回值之外，不要从`SideEffect`返回任何其他数据。

以下示例演示了如何使用`SideEffect`:

```go
encodedRandom := SideEffect(func(ctx workflow.Context) interface{} {
        return rand.Intn(100)
})

var random int
encodedRandom.Get(&random)
if random < 50 {
        ....
} else {
        ....
}
```

