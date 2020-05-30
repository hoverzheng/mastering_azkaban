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
```java
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



### 提交flow

* handleAjaxExecute函数

```java
private void handleAjaxExecute(final HttpServletRequest req,
    final Map<String, Object> respMap, final int execId) {
  try {
    this.flowRunnerManager.submitFlow(execId);
  } catch (final ExecutorManagerException e) {
    logger.error(e.getMessage(), e);
    respMap.put(ConnectorParams.RESPONSE_ERROR, e.getMessage());
  }
}
```

* FlowRunnerManager#submitFlow函数

```java
public void submitFlow(final int execId) throws ExecutorManagerException {
  // 该execId正在执行，则什么都不做直接返回
  if (isAlreadyRunning(execId)) {
    return;
  }
  // 先从数据库中获取一个execId的flow信息，然后创建一个flowRunner
  final FlowRunner runner = createFlowRunner(execId);
  // Check again.
  if (isAlreadyRunning(execId)) {
    return;
  }
  submitFlowRunner(runner);
}
```

（1）在该函数中先判断execId这个flow是否在runningFlows这个concurrentHashMap中。

（2）第7行，从数据库中获取该execId的信息，并创建一个FlowRunner

（3）再次判断execId是否已经在运行

（4）通过submitFlowRunner函数提交runner，其实是把该runner放到正在运行的runningFlows队列中，并把线程提交到线程池中执行。

* createFlowRunner函数

该函数的执行步骤如下：

（1）从数据库中获取一个flow（ExecutableFlow对象）来执行:从表execution_flows中获取exec_id为execId的数据。

（2）创建FlowRunner对象，这是一个线程对象

```java
final FlowRunner runner =
        new FlowRunner(flow, this.executorLoader, this.projectLoader,this.jobtypeManager,
      this.azkabanProps,this.azkabanEventReporter,this.alerterHolder,this.commonMetrics);
```

* submitFlowRunner(runner)函数

```java
private void submitFlowRunner(final FlowRunner runner) throws ExecutorManagerException {
  // 放入正在运行的runner队列中
  this.runningFlows.put(runner.getExecutionId(), runner);
  try {
    // The executorService already has a queue.
    // The submit method below actually returns an instance of FutureTask,
    // which implements interface RunnableFuture, which extends both
    // Runnable and Future interfaces
    // 把runner提交到线程池，提交runner，开始执行
    final Future<?> future = this.executorService.submit(runner);
    // keep track of this future
    // 通过future来获取任务执行的最终结果，把future放到执行完成任务队列中
    this.submittedFlows.put(future, runner.getExecutionId());
    // update the last submitted time.
    // 更新最终提交时间
    this.lastFlowSubmittedDate = System.currentTimeMillis();
	} catch (final RejectedExecutionException re) {
    ···
  }
}
```

（1）第3行，先把FlowRunner对象runner放到运行的runner队列中。

（2）第10行，把runner提交到线程池，开始执行，并返回一个future对象

（3）获取返回的future对象，并把该future对象放到执行完成任务队列中。

（4）更新最终提交时间。



### 取消flow

* handleAjaxCancel函数

```java
private void handleAjaxCancel(final Map<String, Object> respMap, final int execid,
    final String user) {
  if (user == null) {
    respMap.put(ConnectorParams.RESPONSE_ERROR, "user has not been set");
    return;
  }

  try {
    //最终调用cancelFlow函数来取消flow
    this.flowRunnerManager.cancelFlow(execid, user);
    respMap.put(ConnectorParams.STATUS_PARAM, ConnectorParams.RESPONSE_SUCCESS);
  } catch (final ExecutorManagerException e) {
    logger.error(e.getMessage(), e);
    respMap.put(ConnectorParams.RESPONSE_ERROR, e.getMessage());
  }
}
```

（1）第3行，必须要有用户名，也就是说用户必须登录，才能发送取消flow的命令。

（2）调用FlowRunnerManager#cancelFlow函数来取消flow。

* FlowRunnerManager#cancelFlow

```java
public void cancelFlow(final int execId, final String user)
    throws ExecutorManagerException {
  final FlowRunner flowRunner = this.runningFlows.get(execId);
  if (flowRunner == null) {
    throw new ExecutorManagerException("Execution " + execId + " is not running.");
  }
  // account for those unexpected cases where a completed execution remains in the runningFlows
  //collection due to, for example, the FLOW_FINISHED event not being emitted/handled.
  if(Status.isStatusFinished(flowRunner.getExecutableFlow().getStatus())) {
    LOGGER.warn("Found a finished execution in the list of running flows: " + execId);
    throw new ExecutorManagerException("Execution " + execId + " is already finished.");
  }

  flowRunner.kill(user);
}
```

（1）从正在运行的flow队列runningFlows中，通过execId获取FlowRunner对象。

（2）若该FlowRunner对象的状态是已经完成，也就是处于：FAILED、KILLED、SUCCEEDED、SKIPPED、FAILED_SUCCEEDED、CANCELLED，等几个状态中。在抛出异常。

（3）调用flowRunner.kill(user)来取消flow。

* FlowRunner#kill函数

该函数其实调用了FlowRunner#kill()函数，该函数的定义如下：

```java
public void kill() {
  synchronized (this.mainSyncObj) {
    if (this.flowKilled) {
      return;
    }
    this.logger.info("Kill has been called on execution " + this.execId);
    this.flow.setStatus(Status.KILLING);
    // If the flow is paused, then we'll also unpause
    this.flowPaused = false;
    this.flowKilled = true;

    if (this.watcher != null) {
      this.logger.info("Watcher is attached. Stopping watcher.");
      this.watcher.stopWatcher();
      this.logger
          .info("Watcher cancelled status is " + this.watcher.isWatchCancelled());
    }

    this.logger.info("Killing " + this.activeJobRunners.size() + " jobs.");
    for (final JobRunner runner : this.activeJobRunners) {
      runner.kill();
    }
    updateFlow();
  }
  interrupt();
}
```



* JobRunner#kill函数

