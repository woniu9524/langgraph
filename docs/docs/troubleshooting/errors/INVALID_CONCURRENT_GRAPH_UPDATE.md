# INVALID_CONCURRENT_GRAPH_UPDATE

LangGraph 的 [`StateGraph`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.state.StateGraph) 从多个节点接收到对其状态的并发更新，而某个状态属性不支持此操作。

这种情况发生的一个常见原因是，在图中使用了 [fanout](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 或其他并行执行方式，并且图的定义如下：

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

如果上述图中的某个节点返回 `{ "some_key": "some_string_value" }`，这将用 `"some_string_value"` 覆盖 `"some_key"` 的状态值。
然而，如果在单步执行的 fanout 中有多个节点为 `"some_key"` 返回值，图将抛出此错误，因为在如何更新内部状态方面存在不确定性。

为了解决这个问题，您可以定义一个组合多个值的 reducer：

```python hl_lines="5-6"
import operator
from typing import Annotated

class State(TypedDict):
    # operator.add reducer 函数使此成为追加模式
    some_key: Annotated[list, operator.add]
```

这将允许您定义处理从并行执行的多个节点返回的相同键的逻辑。

## 故障排除

以下方法可能有助于解决此错误：

- 如果您的图并行执行节点，请确保您已为相关状态键定义了 reducer。