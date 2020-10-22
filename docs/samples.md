- branch

  异步运行多个活动，最后一起收集结果，类似于 WaitGroup

- cancelactivity

  活动的运行与触发中途取消

- child-workflow

  在工作流中运行子工作流

- child-workflow-continue-as-new

  将子工作流重新作为新工作流多次运行，可以用全局参数控制运行次数

- choice-exclusive

  使用参数控制选择运行哪个活动

- choice-multi

  异步运行多个传入不同参数的活动

- cron

  cron 作业工作流

- ctxpropagation

  使用 ctx 在工作流和活动之间传递数据

- dsl

  dsl内容

- dynamic

  **动态**活动注册和运行

- expense

  订单创建与异步人工消费，活动流等待**活动异步完成**的触发

- fileprocessing

  使用**会话**绑定文件的下载处理上传在同一个 Worker 上完成

- greetings

  收集上一个活动的参数并传入下个活动

- helloworld

  helloworld

- mutex

  使用外部 **signal** 和工作流之间 **singal** 完成一个互斥锁

- parallel

  使用 **workflow.Go** 并发在工作流中运行多套逻辑，每套逻辑可包含不同的活动

- pickfirst

  同时运行多个活动，使用 **selector.Select(ctx)** 等待一个活动完成，并取消剩余的活动

- pso

  PSO算法实现

- query

  主要展示查询 workflow 信息

- recovery

  工作流的执行状态恢复与查询

- retryactivity

  活动的重试机制，包括从失败进度点重试

- searchattributes

  Workflow 筛选器的配置与使用

- splitmerge

  使用 **workflow.Go** 工作流中并发进行块处理，并使用 channel 汇总结果

- timer

  使用计时器和 **selector.Select** 完成活动超时等待