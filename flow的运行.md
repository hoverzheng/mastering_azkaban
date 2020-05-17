# flow的运行

本文介绍azkaban中flow的执行实现逻辑。

## FlowRunner
FlowRunner是一个线程，当提交给线程池时会执行run函数。


run函数的核心代码如下：

```
public void run() {
    try {
      if (this.executorService == null) {
        this.executorService = Executors.newFixedThreadPool(this.numJobThreads);
      }
      setupFlowExecution();
      this.flow.setStartTime(System.currentTimeMillis());

      this.logger.info("Updating initial flow directory.");
      updateFlow();
      this.logger.info("Fetching job and shared properties.");
      if (!FlowLoaderUtils.isAzkabanFlowVersion20(this.flow.getAzkabanFlowVersion())) {
        loadAllProperties();
      }

      this.fireEventListeners(
          Event.create(this, EventType.FLOW_STARTED, new EventData(this.getExecutableFlow())));
      runFlow();
    } catch (final Throwable t) {
        ...
    }
  }
```


### setupFlowExecution


### runFlow