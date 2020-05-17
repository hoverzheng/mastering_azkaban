# FlowRunnerManager实现原理

本文介绍执行器服务的FlowRunnerManager类实现。

## FlowRunnerManager构造函数
### 构造函数
在FlowRunnerManager构造函数中，会启动以下几个服务：
1. CleanerThread
2. PollingService


### PollingService服务
该服务会周期性的从数据库中获取新的执行流，并把这些执行流提交到executor上去执行。

当启动
1. 初始化状态时execId=-1，此时需要更新数据中的executor数据，并获取最新的executor的配置数据。
2. 若execId!=-1，说明不是初始化状态，此时会先从DB中获取execId，若execId != -1，则调用FlowRunnerManager#submitFlow函数来提交flow。
3. submitFlow函数的流程如下：
   * 若execId正在运行，则什么都不做，直接返回。
   * 通过函数FlowRunnerManager#createFlowRunner创建一个FlowRunner对象: runner，该对象的execId参数来自submitFlow的参数。
   * 再次判断execId是否正在运行，若是，直接返回。
   * 通过FlowRunnerManager#submitFlowRunner函数提交runner对象。
   * 
   

#### createFlowRunner




#### submitFlowRunner
该函数的实现逻辑如下：
1. 把runner放到runningFlows这个ConcurrentHashMap中。
2. 把runner提交到线程池中，执行该runner。
3. 为了跟踪已经提交的runner，需要把runner放到submittedFlows的同步map中。
4. 更新最后一个runner的提交时间。


该函数的核心代码如下：
```
  private void submitFlowRunner(final FlowRunner runner) throws ExecutorManagerException {
    // 把runner放到runningFlows这个ConcurrentHashMap的map中
    this.runningFlows.put(runner.getExecutionId(), runner);
    try {
      // 把runner提交到线程池中，执行该runner
      final Future<?> future = this.executorService.submit(runner);
      // keep track of this future
      // 为了跟踪已经提交的runner，需要把runner放到submittedFlows的map中
      this.submittedFlows.put(future, runner.getExecutionId());
      // update the last submitted time.
      // 更新最后一个runner的提交时间
      this.lastFlowSubmittedDate = System.currentTimeMillis();
    } catch (final RejectedExecutionException re) {
    ...
  }
```

此时，runner已经提交到线程池中，若资源足够，会获取一个新的线程，并开始运行runner的线程。