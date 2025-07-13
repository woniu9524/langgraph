# INVALID_GRAPH_NODE_RETURN_VALUE

LangGraph 的 [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 收到了来自节点的非 dict（字典）返回类型。示例如下：

```python
class State(TypedDict):
    some_key: str

def bad_node(state: State):
    # 应返回一个包含 "some_key" 值的 dict，而不是一个列表
    return ["whoops"]

builder = StateGraph(State)
builder.add_node(bad_node)
...

graph = builder.compile()
```

调用上面的图将导致类似以下的错误：

```python
graph.invoke({ "some_key": "someval" });
```

```
InvalidUpdateError: Expected dict, got ['whoops']
For troubleshooting, visit: https://python.langchain.com/docs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

图中的节点必须返回一个包含在你的 state 中定义的键的 dict。

## Troubleshooting（故障排除）

以下方法可能有助您解决此错误：

- 如果您的节点中有复杂的逻辑，请确保所有代码路径都返回您定义的 state 的相应 dict。