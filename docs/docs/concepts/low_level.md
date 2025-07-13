---
search:
  boost: 2
---

# Graph API 概念

## 图

其核心在于，LangGraph 将代理工作流建模为图。您可以使用三个关键组件来定义代理的行为：

1. [`State`](#state)：一个共享的数据结构，代表您应用程序的当前快照。它可以是任何 Python 类型，但通常是 `TypedDict` 或 Pydantic `BaseModel`。

2. [`Nodes`](#nodes)：编码代理逻辑的 Python 函数。它们接收当前 `State` 作为输入，执行一些计算或副作用，并返回更新后的 `State`。

3. [`Edges`](#edges)：根据当前 `State` 确定下一个要执行的 `Node` 的 Python 函数。它们可以是条件分支或固定转换。

通过组合 `Nodes` 和 `Edges`，您可以创建复杂的、循环的工作流，随着时间的推移不断演进 `State`。然而，真正的威力来自于 LangGraph 管理 `State` 的方式。请注意：`Nodes` 和 `Edges` 仅仅是 Python 函数，它们可以包含 LLM 或纯粹的 Python 代码。

简而言之：_节点执行工作，边决定下一步做什么_。

LangGraph 底层的图算法使用 [消息传递](https://en.wikipedia.org/wiki/Message_passing) 来定义通用程序。当一个节点完成其操作时，它会将消息沿着一条或多条边发送到其他节点。这些接收节点随后执行它们的函数，将结果消息传递给下一组节点，过程持续进行。受到 Google 的 [Pregel](https://research.google/pubs/pregel-a-system-for-large-scale-graph-processing/) 系统的启发，程序以离散的“超级步”进行。

一个超级步可以被视为一次对图节点的迭代。并行运行的节点属于同一个超级步，而顺序运行的节点则属于不同的超级步。在图执行开始时，所有节点都处于 `inactive`（非激活）状态。当一个节点在其任何传入边（或“通道”）上接收到新消息（状态）时，它就会变成 `active`（激活）状态。然后，激活的节点运行其函数，并通过更新进行响应。在每个超级步结束时，没有传入消息的节点会通过将自身标记为 `inactive` 来投票决定 `halt`（停止）。当所有节点都处于 `inactive` 状态且没有消息在传输时，图执行终止。

### StateGraph

`StateGraph` 类是主要使用的图类。它由用户定义的 `State` 对象参数化。

### 编译您的图

要构建您的图，您首先需要定义 [state](#state)，然后添加 [nodes](#nodes) 和 [edges](#edges)，最后进行编译。编译您的图具体是什么以及为什么需要它？

编译是一个相当简单的步骤。它对图的结构进行一些基本检查（例如，没有孤立的节点）。它也是您可以指定运行时参数的地方，例如 [checkpointers](./persistence.md) 和断点。您只需调用 `.compile` 方法即可编译您的图：

```python
graph = graph_builder.compile(...)
```

在使用图 **必须** 先编译它。

## State

定义图时，您首先要做的就是定义图的 `State`。`State` 包括 [图的模式](#schema) 以及用于指定如何将更新应用于状态的 [`reducer` 函数](#reducers)。`State` 的模式将是图中所有 `Nodes` 和 `Edges` 的输入模式，它可以是 `TypedDict` 或 Pydantic 模型。所有 `Nodes` 都将发出对 `State` 的更新，然后使用指定的 `reducer` 函数应用这些更新。

### Schema

指定图模式的主要文档化方式是使用 `TypedDict`。但是，我们也支持 [使用 Pydantic BaseModel](../how-tos/graph-api.md#use-pydantic-models-for-graph-state) 作为您的图状态，以添加**默认值**和额外的验证。

默认情况下，图将具有相同的输入和输出模式。如果您想更改这一点，也可以直接指定显式的输入和输出模式。当您有很多键，其中一些明确用于输入，另一些用于输出时，这很有用。请参阅 [此指南](../how-tos/graph-api.md#define-input-and-output-schemas) 以了解如何使用。

#### 多个模式

通常，所有图节点都与单个模式通信。这意味着它们将读取和写入相同的状态通道。但是，有些情况下我们需要更多控制权：

- 内部节点可以传递图中输入/输出不需要的信息。
- 我们也可能希望为图使用不同的输入/输出模式。例如，输出可能只包含一个相关的输出键。

可以将节点写入图内的私有状态通道以进行内部节点通信。我们可以简单地定义一个私有模式 `PrivateState`。有关更多详细信息，请参阅 [此指南](../how-tos/graph-api.md#pass-private-state-between-nodes)。

还可以为图定义显式的输入和输出模式。在这些情况下，我们定义一个包含所有与图操作相关的键的“内部”模式。但是，我们还定义了 `input` 和 `output` 模式，它们是“内部”模式的子集，用于约束图的输入和输出。有关更多详细信息，请参阅 [此指南](../how-tos/graph-api.md#define-input-and-output-schemas)。

让我们看一个例子：

```python
class InputState(TypedDict):
    user_input: str

class OutputState(TypedDict):
    graph_output: str

class OverallState(TypedDict):
    foo: str
    user_input: str
    graph_output: str

class PrivateState(TypedDict):
    bar: str

def node_1(state: InputState) -> OverallState:
    # Write to OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # Read from OverallState, write to PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # Read from PrivateState, write to OutputState
    return {"graph_output": state["bar"] + " Lance"}

builder = StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)
builder.add_node("node_1", node_1)
builder.add_node("node_2", node_2)
builder.add_node("node_3", node_3)
builder.add_edge(START, "node_1")
builder.add_edge("node_1", "node_2")
builder.add_edge("node_2", "node_3")
builder.add_edge("node_3", END)

graph = builder.compile()
graph.invoke({"user_input":"My"})
{'graph_output': 'My name is Lance'}
```

这里有两个微妙且重要的注意事项：

1. 我们将 `state: InputState` 作为输入模式传递给 `node_1`。但是，我们向 `OverallState` 中的一个通道 `foo` 写入。我们如何向不包含在输入模式中的状态通道写入？这是因为节点_可以写入图状态中的任何状态通道_。图状态是初始化时定义的那些状态通道的联合，包括 `OverallState` 以及筛选器 `InputState` 和 `OutputState`。

2. 我们使用 `StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)` 初始化图。那么，我们如何在 `node_2` 中写入 `PrivateState`？图如何获得对该模式的访问权限，如果它没有在 `StateGraph` 初始化中传递的话？我们可以这样做，因为_节点还可以声明其他状态通道_，只要状态模式定义存在。在这种情况下，`PrivateState` 模式已被定义，因此我们可以将 `bar` 添加为图中一个新的状态通道并写入它。

### Reducers

Reducer 是理解节点更新如何应用于 `State` 的关键。`State` 中的每个键都有自己的独立 reducer 函数。如果未显式指定 reducer 函数，则假定所有对该键的更新都应覆盖它。有几种不同类型的 reducer，从默认类型的 reducer 开始：

#### 默认 Reducer

这两个示例展示了如何使用默认 reducer：

**例 A：**

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

在此示例中，未为任何键指定 reducer 函数。假设图的输入是 `{"foo": 1, "bar": ["hi"]}`。然后假设第一个 `Node` 返回 `{"foo": 2}`。这被视为对状态的更新。请注意，`Node` 不需要返回完整的 `State` 模式 - 只需要一个更新。应用此更新后，`State` 将变为 `{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回 `{"bar": ["bye"]}`，那么 `State` 将变为 `{"foo": 2, "bar": ["bye"]}`。

**例 B：**

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

在此示例中，我们使用 `Annotated` 类型为第二个键（`bar`）指定了一个 reducer 函数（`operator.add`）。请注意，第一个键保持不变。假设图的输入是 `{"foo": 1, "bar": ["hi"]}`。然后假设第一个 `Node` 返回 `{"foo": 2}`。这被视为对状态的更新。请注意，`Node` 不需要返回完整的 `State` 模式 - 只需要一个更新。应用此更新后，`State` 将变为 `{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回 `{"bar": ["bye"]}`，那么 `State` 将变为 `{"foo": 2, "bar": ["hi", "bye"]}`。请注意，这里 `bar` 键通过将两个列表相加来更新。

### 在图状态中处理消息

#### 何时使用消息？

大多数现代 LLM 提供商都有一个接受消息列表作为输入的聊天模型接口。LangChain 的 [`ChatModel`](https://python.langchain.com/docs/concepts/#chat-models) 特别接受 `Message` 对象列表作为输入。这些消息有多种形式，例如 `HumanMessage`（用户输入）或 `AIMessage`（LLM 响应）。要详细了解消息对象是什么，请参阅 [此](https://python.langchain.com/docs/concepts/#messages) 概念指南。

#### 在您的图中处理消息

在许多情况下，将先前的对话历史作为消息列表存储在图状态中很有帮助。为此，我们可以向图状态添加一个键（通道），该通道存储 `Message` 对象列表，并用 reducer 函数对其进行注释（参见示例文中 `messages` 键）。Reducer 函数对于告诉图如何使用每次状态更新（例如，当节点发送更新时）来更新状态中的 `Message` 对象列表至关重要。如果您不指定 reducer，每次状态更新都会用最近提供的值覆盖消息列表。如果您只想将消息追加到现有列表中，可以使用 `operator.add` 作为 reducer。

但是，您也可能希望手动更新图状态中的消息（例如，人工干预）。如果您使用 `operator.add`，您发送到图的手动状态更新将被追加到现有消息列表中，而不是更新现有消息。为避免这种情况，您需要一个可以跟踪消息 ID 并覆盖现有消息（如果已更新）的 reducer。要实现这一点，您可以使用预先构建的 `add_messages` 函数。对于全新的消息，它只会追加到现有列表中，但它也会正确处理现有消息的更新。

#### 序列化

除了跟踪消息 ID 外，当在 `messages` 通道上收到状态更新时，`add_messages` 函数还会尝试将消息反序列化为 LangChain `Message` 对象。有关 LangChain 序列化/反序列化的更多信息，请参阅 [此处](https://python.langchain.com/docs/how_to/serialization/）。这允许以以下格式发送图输入/状态更新：

```python
# 支持这种方式
{"messages": [HumanMessage(content="message")]}

# 也支持这种方式
{"messages": [{"type": "human", "content": "message"}]}
```

由于在使用 `add_messages` 时，状态更新始终被反序列化为 LangChain `Messages`，您应该使用点符号来访问消息属性，例如 `state["messages"][-1].content`。下面是一个使用 `add_messages` 作为 reducer 函数的图示例。

```python
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

#### MessagesState

由于在状态中拥有消息列表非常普遍，因此存在一个预先构建的状态 `MessagesState`，它可以轻松地使用消息。`MessagesState` 定义了一个只包含 `messages` 键，该键是一个 `AnyMessage` 对象列表，并使用 `add_messages` reducer。通常，需要跟踪的状态不仅仅是消息，因此我们看到人们继承该状态并添加更多字段，例如：

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

## Nodes

在 LangGraph 中，节点通常是 python 函数（同步或异步），其中**第一个**位置参数是 [state](#state)，可选地，**第二个**位置参数是“config”，其中包含可选的 [可配置参数](#configuration)（例如 `thread_id`）。

类似于 `NetworkX`，您可以使用 [add_node][langgraph.graph.StateGraph.add_node] 方法将这些节点添加到图中：

```python
from typing_extensions import TypedDict

from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph

class State(TypedDict):
    input: str
    results: str

builder = StateGraph(State)


def my_node(state: State, config: RunnableConfig):
    print("In node: ", config["configurable"]["user_id"])
    return {"results": f"Hello, {state['input']}!"}


# 第二个参数是可选的
def my_other_node(state: State):
    return state


builder.add_node("my_node", my_node)
builder.add_node("other_node", my_other_node)
...
```

在后台，函数被转换为 [RunnableLambda](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.base.RunnableLambda.html#langchain_core.runnables.base.RunnableLambda)s，它们为您的函数增加了批处理和异步支持，以及原生的跟踪和调试功能。

如果您将一个节点添加到图中而未指定名称，它将获得一个等效于函数名称的默认名称。

```python
builder.add_node(my_node)
# 然后您可以通过将其引用为 "my_node" 来创建指向/来自此节点的边
```

### `START` Node

`START` Node 是一个特殊节点，代表将用户输入发送到图的节点。引用此节点的主要目的是确定应首先调用哪些节点。

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### `END` Node

`END` Node 是一个特殊节点，代表一个终端节点。当您想指明哪些边完成后没有后续操作时，引用此节点。

```
from langgraph.graph import END

graph.add_edge("node_a", END)
```

### Node Caching

LangGraph 支持基于节点输入的任务/节点缓存。要使用缓存：

* 在编译图（或指定入口点）时指定缓存
* 为节点指定缓存策略。每种缓存策略支持：
    * `key_func` 用于根据节点输入生成缓存键，默认情况下使用 pickle 哈希输入。
    * `ttl`，缓存的生存时间（秒）。如果未指定，缓存将永远不会过期。

例如：

```python
import time
from typing_extensions import TypedDict
from langgraph.graph import StateGraph
from langgraph.cache.memory import InMemoryCache
from langgraph.types import CachePolicy


class State(TypedDict):
    x: int
    result: int


builder = StateGraph(State)


def expensive_node(state: State) -> dict[str, int]:
    # expensive computation
    time.sleep(2)
    return {"result": state["x"] * 2}


builder.add_node("expensive_node", expensive_node, cache_policy=CachePolicy(ttl=3))
builder.set_entry_point("expensive_node")
builder.set_finish_point("expensive_node")

graph = builder.compile(cache=InMemoryCache())

print(graph.invoke({"x": 5}, stream_mode='updates'))  # (1)!
[{'expensive_node': {'result': 10}}]
print(graph.invoke({"x": 5}, stream_mode='updates'))  # (2)!
[{'expensive_node': {'result': 10}, '__metadata__': {'cached': True}}]
```

1. 第一次运行花费了完整的几秒钟（由于模拟的耗时计算）。
2. 第二次运行利用缓存并快速返回。

## Edges

边定义了逻辑的路由方式以及图如何决定停止。这是您的代理工作方式以及不同节点如何相互通信的重要部分。有几种关键的边类型：

- 普通边：直接从一个节点到下一个节点。
- 条件边：调用一个函数来确定下一个要转到的节点。
- 入口点：用户输入到达时首先调用的节点。
- 条件入口点：用户输入到达时调用一个函数来确定首先要调用的节点。

一个节点可以有多条出边。如果一个节点有多条出边，**所有**这些目标节点将在下一个超级步中并行执行。

### 普通边

如果您**总是**想从节点 A 转到节点 B，您可以直接使用 [add_edge][langgraph.graph.StateGraph.add_edge] 方法。

```python
graph.add_edge("node_a", "node_b")
```

### 条件边

如果您想**选择性地**路由到 1 个或多个边（或选择性地终止），您可以使用 [add_conditional_edges][langgraph.graph.StateGraph.add_conditional_edges] 方法。此方法接受一个节点名称和一个在执行该节点后调用的“路由函数”：

```python
graph.add_conditional_edges("node_a", routing_function)
```

与节点类似，`routing_function` 接受图的当前 `state` 并返回值。

默认情况下，`routing_function` 的返回值用作下一个节点名称（或节点列表）。所有这些节点将在下一个超级步中并行运行。

您可以选择提供一个字典，将 `routing_function` 的输出映射到下一个节点名称。

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

!!! tip
    如果您想在单个函数中结合状态更新和路由，请使用 [`Command`](#command) 而不是条件边。

### 入口点

入口点是图启动时首先运行的节点。您可以使用虚拟 [`START`][langgraph.constants.START] 节点到第一个执行节点之间的 [add_edge][langgraph.graph.StateGraph.add_edge] 方法来指定进入图的位置。

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

### 条件入口点

条件入口点允许您根据自定义逻辑从不同的节点开始。您可以使用虚拟 [`START`][langgraph.constants.START] 节点到此来实现 [`add_conditional_edges`][langgraph.graph.StateGraph.add_conditional_edges]。

```python
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

您可以选择提供一个字典，将 `routing_function` 的输出映射到下一个节点名称。

```python
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

## `Send`

默认情况下，`Nodes` 和 `Edges` 会提前定义并操作相同的共享状态。但是，有时精确的边不是提前知道的，或者您可能希望同时存在不同版本的 `State`。一个常见的例子是 [map-reduce](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 设计模式。在此设计模式中，第一个节点可能会生成一个对象列表，并且您可能希望将某些其他节点应用于所有这些对象。对象的数量可能不是提前已知的（这意味着边的数量可能不是已知的），并且下游 `Node` 的输入 `State` 应该不同（每个生成的对象一个）。

为了支持此设计模式，LangGraph 支持从条件边返回 [`Send`][langgraph.types.Send] 对象。`Send` 接收两个参数：第一个是节点名称，第二个是要传递给该节点的状态。

```python
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

## `Command`

将控制流（边）和状态更新（节点）结合起来可能很有用。例如，您可能希望在**同一个节点**中**同时**执行状态更新并决定下一个要转到哪个节点。LangGraph 提供了一种通过从节点函数返回 [`Command`][langgraph.types.Command] 对象的方法：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```

使用 `Command`，您还可以实现动态控制流行为（与 [条件边](#conditional-edges) 相同）：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

!!! important

    在节点函数中返回 `Command` 时，您必须添加返回类型注解，并列出节点路由到的节点名称，例如 `Command[Literal["my_other_node"]]`。这是图渲染所必需的，它告诉 LangGraph `my_node` 可以导航到 `my_other_node`。

请查看此 [操作指南](../how-tos/graph-api.md#combine-control-flow-and-state-updates-with-command) 以获取使用 `Command` 的端到端示例。

### 何时应使用 Command 而不是条件边？

当您需要**同时**更新图状态**并**路由到另一个节点时，请使用 `Command`。例如，在实现[多代理交接](./multi_agent.md#handoffs)时，路由到不同的代理并将某些信息传递给该代理很重要。

使用 [条件边](#conditional-edges) 在不更新状态的情况下有条件地路由节点。

### 导航到父图中的节点

如果您使用[子图](./subgraphs.md)，您可能希望从子图中的节点导航到另一个子图（即父图中的另一个节点）。为此，您可以在 `Command` 中将 `graph=Command.PARENT` 指定为 `Command`:

```python
def my_node(state: State) -> Command[Literal["other_subgraph"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # 其中 `other_subgraph` 是父图中的一个节点
        graph=Command.PARENT
    )
```

!!! note

    将 `graph` 设置为 `Command.PARENT` 将导航到最近的父图。

!!! important "使用 `Command.PARENT` 进行状态更新"

    当您从子图节点向父图节点发送共享父子图 [模式](#schema) 的键的更新时，您**必须**在父图状态中为要更新的键定义一个[reducer](#reducers)。请参阅此 [示例](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph)。

这在实现[多代理交接](./multi_agent.md#handoffs)时特别有用。

请参阅此 [指南](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph) 获取详细信息。

### 在工具中使用

一个常见的用例是从工具内部更新图状态。例如，在客户支持应用程序中，您可能希望在对话开始时根据客户的账号或 ID 查找客户信息。

有关详细信息，请参阅此 [操作指南](../how-tos/graph-api.md#use-inside-tools)。

### 人工干预

`Command` 是人工干预工作流的重要组成部分：在使用 `interrupt()` 收集用户输入时，`Command` 用于提供输入并通过 `Command(resume="User input")` 恢复执行。有关更多信息，请参阅此 [概念指南](./human_in_the_loop.md)。

## Graph Migrations

LangGraph 可以轻松处理图定义（节点、边和状态）的迁移，即使在使用检查点来跟踪状态时也是如此。

- 对于位于图末尾的线程（即未中断的线程），您可以更改图的整个拓扑结构（即所有节点和边，删除、添加、重命名等）。
- 对于当前中断的线程，我们支持除重命名/删除节点之外的所有拓扑更改（因为该线程可能会进入一个不再存在的节点）- 如果这是一个障碍，请联系我们，我们可以优先解决。
- 对于修改状态，我们支持添加和删除键的完全向后和向前兼容性。
- 重命名过的状态键会在现有线程中丢失其保存的状态。
- 类型发生不兼容更改的状态键可能会导致之前发生更改的线程出现问题 - 如果这是一个障碍，请联系我们，我们可以优先解决。

## Configuration

创建图时，您还可以标记图的某些部分是可配置的。这通常是为了方便地切换模型或系统提示。这使您可以创建一个单一的“认知架构”（图），但拥有其多个不同的实例。

创建图时，您可以选择性地指定 `config_schema`。

```python
class ConfigSchema(TypedDict):
    llm: str

graph = StateGraph(State, config_schema=ConfigSchema)
```

然后，您可以使用 `configurable` config 字段将此配置传递到图中。

```python
config = {"configurable": {"llm": "anthropic"}}

graph.invoke(inputs, config=config)
```

然后，您可以在节点或条件边内访问和使用此配置：

```python
def node_a(state, config):
    llm_type = config.get("configurable", {}).get("llm", "openai")
    llm = get_llm(llm_type)
    ...
```

有关配置的完整 breakdown，请参阅此 [操作指南](../how-tos/graph-api.md#add-runtime-configuration)。

### Recursion Limit

递归限制设置了图在单次执行期间可以执行的[超级步](#graphs)的最大数量。一旦达到限制，LangGraph 将引发 `GraphRecursionError`。默认情况下，此值为 25 步。递归限制可以在运行时设置为任何图，并通过 config 字典传递给 `.invoke`/`.stream`。重要的是，`recursion_limit` 是一个独立的 `config` 键，不应像其他所有用户定义的配置一样传递到 `configurable` 键内。请参见下面的示例：

```python
graph.invoke(inputs, config={"recursion_limit": 5, "configurable":{"llm": "anthropic"}})
```

阅读此[操作指南](https://langchain-ai.github.io/langgraph/how-tos/recursion-limit/)以了解更多关于递归限制如何工作的信息。

## Visualization

能够可视化图通常很好，尤其是在它们变得更复杂时。LangGraph 提供了几种可视化图的内置方法。有关更多信息，请参阅此[操作指南](../how-tos/graph-api.md#visualize-your-graph)。