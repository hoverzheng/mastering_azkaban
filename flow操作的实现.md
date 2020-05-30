1.flow操作的实现原理



2.执行一个flow

3.通过REST API触发执行

4.接收执行命令

执行一个flow的REST命令是：

```http
 /executor?ajax=executeFlow
```

当发送该命令时，会在ExecutorServlet#handleAJAXAction函数中处理该请求。处理该请求的代码如下：

```java
      if (ajaxName.equals("executeFlow")) {
        // 处理执行flow的REST API命令
        ajaxExecuteFlow(req, resp, ret, session.getUser());
      }
```

在函数ajaxExecuteFlow中，会获取该flow的id和相关参数，并调用ExecutionController#submitExecutableFlow函数来具体处理执行flow的请求。该函数其实只是把mysql对应的flow的状态修改成active状态。

在执行器（Executor）端，会通过轮训的机制来获取到需要执行的flow，并执行这些flow。





3.取消一个flow

```java
package azkaban.webapp.servlet.ExecutorServlet#handleAJAXAction

private void handleAJAXAction(final HttpServletRequest req,
                              final HttpServletResponse resp, final Session session) {
  ...
 else if (ajaxName.equals("cancelFlow")) {
  ajaxCancelFlow(req, resp, ret, session.getUser(), exFlow);
..
}
```

接下来调用：

```
this.executorManagerAdapter.cancelFlow(exFlow, user.getUserId());
```

​	接下来调用：

```
ExecutionController#cancelFlow
```

接下来发送 命令：

```java
this.apiGateway
    .callWithReferenceByUser(pair.getFirst(), ConnectorParams.CANCEL_ACTION, userId);
```



接下来，在Executor端：execapp包中的ExecutorServlet会接受该请求（ConnectorParams.CANCEL_ACTION），并解析参数，调用handleRequest函数来处理该请求。

```java
package azkaban.execapp.ExecutorServlet

public void 
handleRequest(final HttpServletRequest req, final HttpServletResponse resp) {
	...	
      } else if (action.equals(ConnectorParams.CANCEL_ACTION)) {
            logger.info("Cancel called.");
            // 取消一个flow
            handleAjaxCancel(respMap, execid, user);
  ...          
}

```



