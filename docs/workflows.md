# 工作流

## 概述

Temporal 的核心抽象是一个**故障无感知的有状态的工作流**。工作流代码的状态，包括它创建的局部变量和线程在内，都不受进程和 Temporal 服务故障的影响。这是一个非常强大的概念，因为它封装了状态、工作线程、持久化计时器和事件处理程序。

## 例子

让我们看一个用例。客户注册有试用期的应用程序。在此试用期之后，如果客户尚未取消，则应每月向他收取一次续订费用。必须通过电子邮件通知客户有关费用，并且应该能够随时取消订阅。

这个用例的业务逻辑不是很复杂，可以用几十行代码来表示。但是任何实际的实现都必须确保业务流程是容错的且可扩展的。设计这种系统的方法有很多种。

一种方法是围绕数据库来展开。应用程序进程将定期扫描数据库表中处于特定状态的客户，执行必要的操作，并更新其状态以反映该状态。尽管这个方法是可行的，但是该方法具有很多缺点。最明显的是客户状态的状态机很快会变得极为复杂。例如从信用卡充值或发送电子邮件可能会由于下游系统不可用而失败。失败的调用可能需要长时间的重试，同时最好使用指数增长的重试策略。调用频率应该被限制以免外部系统过载。如果由于某种原因而无法处理单个客户的记录，则应该支持放弃任务以避免阻塞整个过程。基于数据库的方法通常也存在性能问题。

另一种常用的方法是使用计时器服务和队列。任何更新都将推送到队列，然后从中消费数据的工作程序将更新数据库，并可能在下游队列中推送更多消息。对于需要调度的操作可以使用外部计时器服务。这种方法通常有更好的可扩展性，因为数据库不会不断地轮询更改。但这使编程模型更加复杂且容易出错，因为通常在队列系统和数据库之间没有事务级别的更新。

使用 Temporal 可以将整个逻辑封装在一个简单的持久化函数中，该函数直接实现业务逻辑。由于该函数是有状态的，因此实现者无需使用任何其他系统来确保耐用性和容错能力。

这是实现订阅管理用例的示例工作流。它是 Java 语言编写的，但也支持 Go。Python 和 .NET 库正在积极开发中。

``` java
public interface SubscriptionWorkflow {
    @WorkflowMethod
    void execute(String customerId);
}

public class SubscriptionWorkflowImpl implements SubscriptionWorkflow {

  private final SubscriptionActivities activities =
      Workflow.newActivityStub(SubscriptionActivities.class);

  @Override
  public void execute(String customerId) {
    activities.sendWelcomeEmail(customerId);
    try {
      boolean trialPeriod = true;
      while (true) {
        Workflow.sleep(Duration.ofDays(30));
        activities.chargeMonthlyFee(customerId);
        if (trialPeriod) {
          activities.sendEndOfTrialEmail(customerId);
          trialPeriod = false;
        } else {
          activities.sendMonthlyChargeEmail(customerId);
        }
      }
    } catch (CancellationException e) {
      activities.processSubscriptionCancellation(customerId);
      activities.sendSorryToSeeYouGoEmail(customerId);
    }
  }
}
```

再次注意，该代码直接实现了业务逻辑。即使任何其调用的操作（又称活动）会花费很长的时间，代码也是一样的。如果下游处理服务中断了一天，在`chargeMonthlyFee`处阻塞那么长时间也是没问题的。同样，在工作流代码中阻塞睡眠30天也是正常操作。

Temporal 实际上对打开的工作流实例的数量没有可伸缩性限制。因此即使您的站点有数亿消费者，上述代码也同样适用。

开发人员在学习 Temporal 时提出的常见问题是“如何处理我的工作流中的 Worker 进程失败/重新启动”？答案是你不需要。**工作流代码完全不关心 Worker 的任何故障和停机时间，甚至都不关心 Temporal 服务本身**。一旦它们被恢复并且工作流需要处理某些事件（例如计时器或活动完成），工作流就会完全恢复当时的状态并且继续执行。工作流失败的唯一原因是工作流业务代码引发异常，而不是基础架构宕机。

另一个常见问题是工作人员是否可以处理比其缓存大小或支持的线程数更多的工作流实例。答案是处于阻塞状态的工作流可以安全地从 Worker 中删除。之后如果需要（以外部事件的形式）时，可以在其他或相同的 Worker 上重新调用它。因此如果一个 Worker 性能足够，则它可以处理数百万个打开的工作流。

## 状态恢复与决定论

工作流状态恢复利用了事件源，它对代码的编写方式施加了一些限制。主要限制是工作流代码必须是确定性的，这意味着如果多次执行必须产生完全相同的结果。这会排除工作流代码中的所有外部 API 调用，因为外部调用可能会间歇性失败或随时更改其输出。这就是为什么与外界的所有交流都应该通过活动进行的原因。出于相同的原因，工作流代码必须使用 Temporal API 来获取当前时间、睡眠和创建新线程。

要了解 Temporal  执行模型以及恢复机制，请观看以下视频。涵盖恢复部分的动画从 15:50 开始。

<iframe width="807" height="454" src="https://www.youtube.com/embed/qce_AqCkFys" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe>

## ID 唯一性

启动工作流时，工作流 ID 由客户端分配。通常是业务级别 ID，例如客户 ID 或订单 ID。

Temporal 保证在任何时间每个[命名空间](https://docs.temporal.io/docs/learn-glossary/#namespace)只能打开一个具有给定 ID 的工作流（所有工作流类型）。尝试启动使用相同 ID 的工作流将失败，并返回`WorkflowExecutionAlreadyStarted`错误。

如果尝试启动存在具有相同 ID 的完整工作流，则取决于以下`WorkflowIdReusePolicy`选项：

- `AllowDuplicateFailedOnly` 表示仅当先前执行的具有相同 ID 的工作流失败时，才允许启动工作流。
- `AllowDuplicate` 表示允许它不依赖于先前的工作流完成状态而启动。
- `RejectDuplicate` 表示完全不允许使用相同的工作流 ID 启动工作流执行。

默认值为`AllowDuplicateFailedOnly`。

为了区分具有相同工作流ID的工作流的多次运行，Temporal 通过两个 ID 标识工作流：`Workflow Id` 和 `Run Id`。`Run Id`是服务分配的 UUID。确切地说，任何工作流由一个三元组：`Namespace`，`Workflow Id`和`Run Id` 来唯一标识。

## 子工作流

一个工作流可以将其他工作流作为子工作流`child workflows`执行。子级工作流完成或失败会报告给其父工作流。

使用子工作流的一些原因是：

- 子工作流可以由不包含父工作流代码的一组单独的 Worker 托管。因此，它将充当可以从多个其他工作流中调用的单独服务。
- 单个工作流的大小有限。例如它不能执行10万个活动。子工作流可用于将问题分成较小的块。父工作流中有 1000 个子工作流，每个子工作流执行 1000 个活动，就是一百万个执行的活动。
- 子工作流可以使用其 ID 来管理某些资源，以保证唯一性。例如，管理主机升级的工作流可以每个主机有一个子工作流（主机名是工作流 ID），并使用它们来确保主机上的所有操作都是序列化的。
- 子工作流可用于执行某些定期逻辑，而不会增加父工作流历史记录的大小。当父工作流启动一个子工作流时，它会执行周期性的逻辑调用，该逻辑调用根据需要连续执行多次直至完成。如果从父工作流视角来看，它只是一个子工作流调用。

与在单个工作流中囊括所有应用程序逻辑相比，子工作流的主要限制是缺少共享的状态。父工作流和子工作流只能通过异步信号进行通信。但是如果它们之间存在紧密的耦合，那么使用单个工作流并仅依赖共享对象状态可能会更简单。

如果您的问题在已执行活动和已处理信号的数量方面受到限制，我们建议从实现单个工作流开始。它比多个异步通信的工作流更直接。

## 流程超时

通常情况下，限制特定工作流可以运行的时间是很有必要的。为此，工作流提供以下三个参数作为选项：

- `WorkflowExecutionTimeout`允许工作流运行的最长时间，包括重试并作为新的工作流运行。使用`WorkflowRunTimeout`来限制单次运行的执行时间。
- `WorkflowRunTimeout` 允许单个工作流运行的最长时间。
- `WorkflowTaskTimeout`从 Worker 拉出任务的时间点开始处理工作流任务的超时时间。如果决定任务丢失，则会在此超时后重试。

## 工作流重试

工作流代码不受基础架构层次的停机时间和故障影响。但是它仍然可能由于业务逻辑层次的故障而失败。例如，活动可能由于超过重试间隔而失败，并且该错误无法由应用程序代码处理，或者工作流代码具有错误。

某些工作流需要保证即使出现此类故障也可以继续运行。为了支持此类用例，可以在启动工作流时指定可选的指数*重试策略*。如果指定了此选项，则工作流失败后将在计算的重试间隔后从头开始重新启动工作流。以下是重试策略参数：

- `InitialInterval` 第一次重试的退避间隔。
- `BackoffCoefficient`重试策略是指数的。该系数指定重试间隔的增长速度。系数1表示重试间隔始终等于`InitialInterval`。
- `MaximumInterval`指定重试之间的最大间隔。对于大于1的系数有用。
- `MaximumAttempts`指定在出现故障时尝试执行工作流的次数。如果超过此限制，则工作流将失败，并且不会重试。
- `NonRetryableErrorReasons`允许指定不应重试的错误。例如，在某些情况下，对无效参数错误重试没有任何意义。