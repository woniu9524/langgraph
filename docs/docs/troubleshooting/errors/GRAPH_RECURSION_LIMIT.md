# GRAPH_RECURSION_LIMIT

您的 LangGraph [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 在达到停止条件之前，已经达到了最大步数。
这通常是由于代码出现无限循环，例如下面的示例：

:::python

```python
class State(TypedDict):
    some_key: str

builder = StateGraph(State)
builder.add_node("a", ...)
builder.add_node("b", ...)
builder.add_edge("a", "b")
builder.add_edge("b", "a")
...

graph = builder.compile()
```

:::

:::js

```typescript
import { StateGraph } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  someKey: z.string(),
});

const builder = new StateGraph(State)
  .addNode("a", ...)
  .addNode("b", ...)
  .addEdge("a", "b")
  .addEdge("b", "a")
  ...

const graph = builder.compile();
```

:::

然而，复杂的图本身也可能达到默认的限制。

## 故障排除

- 如果您不希望您的图经过很多次迭代，那么您很可能遇到了循环。请检查您的逻辑是否存在无限循环。

:::python

- 如果您使用的是一个复杂的图，您可以在调用图时，通过向 `config` 对象传递一个更高的 `recursion_limit` 值来实现，如下所示：

```python
graph.invoke({...}, {"recursion_limit": 100})
```

:::

:::js

- 如果您使用的是一个复杂的图，您可以在调用图时，通过向 `config` 对象传递一个更高的 `recursionLimit` 值来实现，如下所示：

```typescript
await graph.invoke({...}, { recursionLimit: 100 });
```

:::