# INVALID_CONCURRENT_GRAPH_UPDATE

LangGraph 的 [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 从多个节点接收到对某个不支持此操作的状态属性的并发更新。

这种情况的一种可能发生方式是，如果你的图中使用到了 [fanout](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 或其他并行执行方式，并且你定义了如下所示的图：

:::python

```python hl_lines="2"
class State(TypedDict):
    some_key: str

def node(state: State):
    return {"some_key": "some_string_value"}

def other_node(state: State):
    return {"some_key": "some_string_value"}


builder = StateGraph(State)
builder.add_node(node)
builder.add_node(other_node)
builder.add_edge(START, "node")
builder.add_edge(START, "other_node")
graph = builder.compile()
```

:::

:::js

```typescript hl_lines="2"
import { StateGraph, Annotation, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  someKey: z.string(),
});

const builder = new StateGraph(State)
  .addNode("node", (state) => {
    return { someKey: "some_string_value" };
  })
  .addNode("otherNode", (state) => {
    return { someKey: "some_string_value" };
  })
  .addEdge(START, "node")
  .addEdge(START, "otherNode");

const graph = builder.compile();
```

:::

:::python
如果上述图中的某个节点返回 `{ "some_key": "some_string_value" }`，这将用 `"some_string_value"` 覆盖 `"some_key"` 的状态值。
但是，如果在单个步骤中的并行执行（例如 fanout）中，多个节点都返回了 `"some_key"` 的值，那么图将抛出此错误，因为
对于如何更新内部状态存在不确定性。
:::

:::js
如果上述图中的某个节点返回 `{ someKey: "some_string_value" }`，这将用 `"some_string_value"` 覆盖 `someKey` 的状态值。
但是，如果在单个步骤中的并行执行（例如 fanout）中，多个节点都返回了 `someKey` 的值，那么图将抛出此错误，因为
对于如何更新内部状态存在不确定性。
:::

要解决这个问题，你可以定义一个可以合并多个值的 reducer：

:::python

```python hl_lines="5-6"
import operator
from typing import Annotated

class State(TypedDict):
    # operator.add reducer 函数使此键变为追加模式
    some_key: Annotated[list, operator.add]
```

:::

:::js

```typescript hl_lines="4-7"
import { withLangGraph } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  someKey: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (existing, update) => existing.concat(update),
    },
    default: () => [],
  }),
});
```

:::

这将允许你定义逻辑来处理从并行执行的多个节点返回的相同键。

## 故障排查

以下方法可能有助于解决此错误：

- 如果你的图并行执行节点，请确保你已为相关的状态键定义了 reducer。