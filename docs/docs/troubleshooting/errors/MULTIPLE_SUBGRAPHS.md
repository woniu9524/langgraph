# 多子图

您正在使用 checkpointing（检查点）为每个子图在单个 LangGraph 节点内多次调用子图。

由于子图的检查点命名空间工作方式存在内部限制，目前不允许这样做。

## 故障排除

以下方法可能有助于解决此错误：

:::python

- 如果您不需要中断/恢复子图，可以在编译时传递 `checkpointer=False`，如下所示：`.compile(checkpointer=False)`
  :::

:::js

- 如果您不需要中断/恢复子图，可以在编译时传递 `checkpointer: false`，如下所示：`.compile({ checkpointer: false })`
  :::

- 不要在同一节点中多次调用图，而是使用 [`Send`](https://langchain-ai.github.io/langgraph/concepts/low_level/#send) API。