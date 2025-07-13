# Multiple Subgraphs

您在一个启用了 checkpointing 的 LangGraph 节点中多次调用子图。

由于子图的 checkpoint 名称空间工作方式存在内部限制，目前不允许这样做。

## 故障排除

以下方法可能有助解决此错误：

- 如果您不需要中断/恢复子图，请在编译子图时传递 `checkpointer=False`，如下所示：`.compile(checkpointer=False)`
- 不要在一个节点中命令式地多次调用图，而是使用 [`Send`](https://langchain-ai.github.io/langgraph/concepts/low_level/#send) API。