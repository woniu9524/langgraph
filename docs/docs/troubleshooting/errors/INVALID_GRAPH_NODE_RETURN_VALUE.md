# INVALID_GRAPH_NODE_RETURN_VALUE

:::python
LangGraph 的 [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 从一个节点接收到了非 dict 类型的返回值。下面是一个例子：

```python
class State(TypedDict):
    some_key: str

def bad_node(state: State):
    # 应该返回一个包含 "some_key" 值的 dict，而不是一个列表
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
InvalidUpdateError: 期望 dict，但得到 ['whoops']
如需排查，请访问：https://python.langchain.com/docs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

图中的节点必须返回一个 dict，其中包含你状态中定义的键。
:::

:::js
LangGraph 的 [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 从一个节点接收到了非 object 类型的返回值。下面是一个例子：

```typescript
import { z } from "zod";
import { StateGraph } from "@langchain/langgraph";

const State = z.object({
  someKey: z.string(),
});

const badNode = (state: z.infer<typeof State>) => {
  // 应该返回一个包含 "someKey" 值的 object，而不是一个数组
  return ["whoops"];
};

const builder = new StateGraph(State).addNode("badNode", badNode);
// ...

const graph = builder.compile();
```

调用上面的图将导致类似以下的错误：

```typescript
await graph.invoke({ someKey: "someval" });
```

```
InvalidUpdateError: 期望 object，但得到 ['whoops']
如需排查，请访问：https://langchain-ai.github.io/langgraphjs/troubleshooting/errors/INVALID_GRAPH_NODE_RETURN_VALUE
```

图中的节点必须返回一个 object，其中包含你状态中定义的键。
:::

## 排查

以下方法可能有助于解决此错误：

:::python

- 如果你的节点中有复杂的逻辑，请确保所有代码路径都返回适合你定义状态的 dict。
  :::

:::js

- 如果你的节点中有复杂的逻辑，请确保所有代码路径都返回适合你定义状态的 object。
  :::