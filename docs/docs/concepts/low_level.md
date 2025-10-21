---
search:
  boost: 2
---

# Graph API 概念

## Graph（图）

LangGraph 的核心是将 agent 工作流抽象为图（graph）。你可以使用三个关键组件来定义 agent 的行为：

1. [`State`](#state)：一个共享的数据结构，代表应用程序的当前状态快照。它可以是任何数据类型，但通常使用共享状态模式（shared state schema）来定义。

2. [`Nodes`](#nodes)：编码 agent 逻辑的函数。它们接收当前状态作为输入，执行一些计算或副作用，并返回更新后的状态。

3. [`Edges`](#edges)：根据当前状态决定下一个要执行的 `Node` 的函数。它们可以是条件分支或固定转换。

通过组合 `Nodes` 和 `Edges`，你可以创建复杂的、循环的工作流，随时间推移演进状态。但真正的威力在于 LangGraph 如何管理这个状态。强调一下：`Nodes` 和 `Edges` 本质上就是函数——它们可以包含 LLM，也可以只包含纯代码。

简而言之：_节点执行工作，边指引下一步方向_。

LangGraph 的底层图算法使用 [消息传递](https://en.wikipedia.org/wiki/Message_passing) 来定义一个通用的程序。当一个节点完成其操作时，它会将消息沿着一个或多个边发送给其他节点。随后，这些接收节点会执行它们的函数，将产生消息传递给下一组节点，以此类推。受到 Google [Pregel](https://research.google.com/pubs/pregel-a-system-for-large-scale-graph-processing/) 系统的启发，程序按离散的“超级步”（super-steps）进行。

一个超级步可以看作是对图节点的单次迭代。并行运行的节点属于同一超级步，而顺序运行的节点则属于不同的超级步。在图执行开始时，所有节点都处于 `inactive`（非活动）状态。当一个节点在其任何传入边（或“通道”）上接收到新消息（状态）时，它会变为 `active`（活动）状态。活动节点随后运行其函数并返回更新。在每个超级步结束时，没有接收到消息并且投票决定 `halt`（停止）的节点会被标记为 `inactive`。当所有节点都处于 `inactive` 状态且没有消息在传输时，图执行终止。

### StateGraph

`StateGraph` 类是主要的图类。它通过用户定义的 `State` 对象进行参数化。

### 编译你的图

要构建你的图，首先需要定义 [state](#state)，然后添加 [nodes](#nodes) 和 [edges](#edges)，最后进行编译。到底什么是编译你的图，为什么需要它？

编译是一个相当简单的步骤。它会对图的结构进行一些基本的检查（例如，没有孤立的节点等）。你也可以在这里指定运行时参数，如 [checkpointers](./persistence.md) 和断点。通过调用 `.compile` 方法即可编译你的图：

:::python

```python
graph = graph_builder.compile(...)
```

:::

:::js

```typescript
const graph = new StateGraph(StateAnnotation)
  .addNode("nodeA", nodeA)
  .addEdge(START, "nodeA")
  .addEdge("nodeA", END)
  .compile();
```

:::

在你可以使用它**之前，必须**编译你的图。

## State（状态）

:::python
定义图时，首先要定义图的 `State`。`State` 由 [图的 schema](#schema) 以及 [`reducer` 函数](#reducers) 组成，这些函数指定如何将更新应用于状态。`State` 的 schema 将是图中所有 `Nodes` 和 `Edges` 的输入 schema，可以是 `TypedDict` 或 `Pydantic` 模型。所有 `Nodes` 将发出更新到 `State`，然后通过指定的 `reducer` 函数应用这些更新。
:::

:::js
定义图时，首先要定义图的 `State`。`State` 由 [图的 schema](#schema) 以及 [`reducer` 函数](#reducers) 组成，这些函数指定如何将更新应用于状态。`State` 的 schema 将是图中所有 `Nodes` 和 `Edges` 的输入 schema，可以是 Zod schema 或使用 `Annotation.Root` 构建的 schema。所有 `Nodes` 将发出更新到 `State`，然后通过指定的 `reducer` 函数应用这些更新。
:::

### Schema（模式）

:::python
指定图 schema 的主要文档化方式是使用 [`TypedDict`](https://docs.python.org/3/library/typing.html#typing.TypedDict)。如果你想在状态中提供默认值，请使用 [`dataclass`](https://docs.python.org/3/library/dataclasses.html)。如果你需要递归数据验证，我们也支持使用 Pydantic [BaseModel](../how-tos/graph-api.md#use-pydantic-models-for-graph-state) 作为你的图状态（但请注意，pydantic 的性能低于 `TypedDict` 或 `dataclass`）。

默认情况下，图的输入和输出 schema 是相同的。如果你想改变这一点，也可以直接指定显式的输入和输出 schema。当你有很多键，其中一些明确用于输入，另一些用于输出时，这会很有用。请参阅 [此指南](../how-tos/graph-api.md#define-input-and-output-schemas) 以了解如何使用。
:::

:::js
指定图 schema 的主要文档化方式是使用 Zod schemas。然而，我们也支持使用 `Annotation` API 来定义图的 schema。

默认情况下，图的输入和输出 schema 是相同的。如果你想改变这一点，也可以直接指定显式的输入和输出 schema。当你有很多键，其中一些明确用于输入，另一些用于输出时，这会很有用。
:::

#### Multiple schemas（多个 schema）

通常，所有图节点都与单个 schema 进行通信。这意味着它们将读写相同的状态通道。但是，在某些情况下，我们需要更精细地控制：

- 内部节点可以传递图的输入/输出不需要的信息。
-我们也可能希望为图使用不同的输入/输出 schema。例如，输出可能只包含一个相关的输出键。

可以在图内部的私有状态通道中传递信息，以实现内部节点通信。我们可以简单地定义一个私有 schema，`PrivateState`。

也可以为图定义显式的输入和输出 schema。在这种情况下，我们定义一个“内部” schema，其中包含 _所有_ 与图操作相关的键。但是，我们还定义了 `input` 和 `output` schema，它们是“内部” schema 的子集，用于约束图的输入和输出。有关更多详细信息，请参阅 [此指南](../how-tos/graph-api.md#define-input-and-output-schemas)。

让我们看一个例子：

:::python

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
    # 写入 OverallState
    return {"foo": state["user_input"] + " name"}

def node_2(state: OverallState) -> PrivateState:
    # 从 OverallState 读取，写入 PrivateState
    return {"bar": state["foo"] + " is"}

def node_3(state: PrivateState) -> OutputState:
    # 从 PrivateState 读取，写入 OutputState
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
# {'graph_output': 'My name is Lance'}
```

:::

:::js

```typescript
const InputState = z.object({
  userInput: z.string(),
});

const OutputState = z.object({
  graphOutput: z.string(),
});

const OverallState = z.object({
  foo: z.string(),
  userInput: z.string(),
  graphOutput: z.string(),
});

const PrivateState = z.object({
  bar: z.string(),
});

const graph = new StateGraph({
  state: OverallState,
  input: InputState,
  output: OutputState,
})
  .addNode("node1", (state) => {
    // 写入 OverallState
    return { foo: state.userInput + " name" };
  })
  .addNode("node2", (state) => {
    // 从 OverallState 读取，写入 PrivateState
    return { bar: state.foo + " is" };
  })
  .addNode(
    "node3",
    (state) => {
      // 从 PrivateState 读取，写入 OutputState
      return { graphOutput: state.bar + " Lance" };
    },
    { input: PrivateState }
  )
  .addEdge(START, "node1")
  .addEdge("node1", "node2")
  .addEdge("node2", "node3")
  .addEdge("node3", END)
  .compile();

await graph.invoke({ userInput: "My" });
// { graphOutput: 'My name is Lance' }
```

:::

这里有两点需要注意，它们微妙但很重要：

:::python

1. 我们将 `state: InputState` 作为 `node_1` 的输入 schema 传递。但是，我们向 `OverallState` 中的 `foo` 通道写入。我们如何向输入 schema 中未包含的状态通道写入？这是因为节点 _可以写入图状态中的任何状态通道_。图状态是在初始化时定义的、包含 `OverallState` 以及 `InputState` 和 `OutputState` 过滤器进行联合的状态通道。

2. 我们使用 `StateGraph(OverallState,input_schema=InputState,output_schema=OutputState)` 来初始化图。那么，我们如何在 `node_2` 中写入 `PrivateState`？如果 `PrivateState` schema 没有在 `StateGraph` 初始化时传入，图如何获得对它的访问权限？我们可以这样做，因为 _节点也可以声明额外的状态通道_，只要该状态 schema 定义存在。在这种情况下，`PrivateState` schema 定义了，所以我们可以将 `bar` 添加为图中的一个新状态通道并向其写入。
   :::

:::js

1. 我们将 `state` 作为 `node1` 的输入 schema 传递。但是，我们向 `OverallState` 中的 `foo` 通道写入。我们如何向输入 schema 中未包含的状态通道写入？这是因为节点 _可以写入图状态中的任何状态通道_。图状态是在初始化时定义的、包含 `OverallState` 以及 `InputState` 和 `OutputState` 过滤器进行联合的状态通道。

2. 我们使用 `StateGraph({ state: OverallState, input: InputState, output: OutputState })` 来初始化图。那么，我们如何在 `node2` 中写入 `PrivateState`？如果 `PrivateState` schema 没有在 `StateGraph` 初始化时传入，图如何获得对它的访问权限？我们可以这样做，因为 _节点也可以声明额外的状态通道_，只要该状态 schema 定义存在。在这种情况下，`PrivateState` schema 定义了，所以我们可以将 `bar` 添加为图中的一个新状态通道并向其写入。
   :::

### Reducers（归约函数）

Reducers 是理解节点更新如何应用于 `State` 的关键。`State` 中的每个键都有自己的独立 reducer 函数。如果未显式指定 reducer 函数，则假定对该键的所有更新都应覆盖它。有几种不同类型的 reducer，从默认类型的 reducer 开始：

#### Default Reducer（默认 Reducer）

这两个示例展示了如何使用默认 reducer：

**Example A:**

:::python

```python
from typing_extensions import TypedDict

class State(TypedDict):
    foo: int
    bar: list[str]
```

:::

:::js

```typescript
const State = z.object({
  foo: z.number(),
  bar: z.array(z.string()),
});
```

:::

在此示例中，未为任何键指定 reducer 函数。假设图的输入是：

:::python
`{"foo": 1, "bar": ["hi"]}`。然后假设第一个 `Node` 返回 `{"foo": 2}`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` schema——只需更新即可。应用此更新后，`State` 将变为 `{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回 `{"bar": ["bye"]}`，则 `State` 将变为 `{"foo": 2, "bar": ["bye"]}`。
:::

:::js
`{ foo: 1, bar: ["hi"] }`。然后假设第一个 `Node` 返回 `{ foo: 2 }`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` schema——只需更新即可。应用此更新后，`State` 将变为 `{ foo: 2, bar: ["hi"] }`。如果第二个节点返回 `{ bar: ["bye"] }`，则 `State` 将变为 `{ foo: 2, bar: ["bye"] }`。
:::

**Example B:**

:::python

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

在此示例中，我们使用了 `Annotated` 类型为第二个键 (`bar`) 指定了 reducer 函数（`operator.add`）。请注意，第一个键保持不变。假设图的输入是 `{"foo": 1, "bar": ["hi"]}`。然后假设第一个 `Node` 返回 `{"foo": 2}`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` schema——只需更新即可。应用此更新后，`State` 将变为 `{"foo": 2, "bar": ["hi"]}`。如果第二个节点返回 `{"bar": ["bye"]}`，则 `State` 将变为 `{"foo": 2, "bar": ["hi", "bye"]}`。请注意，此时 `bar` 键是通过将两个列表相加来更新的。
:::

:::js

```typescript
import { z } from "zod";
import { withLangGraph } from "@langchain/langgraph/zod";

const State = z.object({
  foo: z.number(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
  }),
});
```

在此示例中，我们使用 `withLangGraph` 函数为第二个键 (`bar`) 指定了 reducer 函数。请注意，第一个键保持不变。假设图的输入是 `{ foo: 1, bar: ["hi"] }`。然后假设第一个 `Node` 返回 `{ foo: 2 }`。这被视为对状态的更新。请注意，`Node` 不需要返回整个 `State` schema——只需更新即可。应用此更新后，`State` 将变为 `{ foo: 2, bar: ["hi"] }`。如果第二个节点返回 `{ bar: ["bye"] }`，则 `State` 将变为 `{ foo: 2, bar: ["hi", "bye"] }`。请注意，此时 `bar` 键是通过将两个数组连接起来更新的。
:::

### Working with Messages in Graph State（在图状态中使用消息）

#### Why use messages?（为什么使用消息？）

:::python
大多数现代 LLM 提供商都有一个接受消息列表作为输入的聊天模型接口。LangChain 的 [`ChatModel`](https://python.langchain.com/docs/concepts/#chat-models) 特别接受 `Message` 对象列表作为输入。这些消息有多种形式，例如 `HumanMessage`（用户输入）或 `AIMessage`（LLM 响应）。要了解有关消息对象的更多信息，请参阅 [此](https://python.langchain.com/docs/concepts/#messages) 概念指南。
:::

:::js
大多数现代 LLM 提供商都有一个接受消息列表作为输入的聊天模型接口。LangChain 的 [`ChatModel`](https://js.langchain.com/docs/concepts/#chat-models) 特别接受 `Message` 对象列表作为输入。这些消息有多种形式，例如 `HumanMessage`（用户输入）或 `AIMessage`（LLM 响应）。要了解有关消息对象的更多信息，请参阅 [此](https://js.langchain.com/docs/concepts/#messages) 概念指南。
:::

#### Using Messages in your Graph（在你的图中使用的消息）

:::python
在许多情况下，将先前的对话历史作为消息列表存储在图状态中会很有帮助。为此，我们可以向图状态添加一个键（通道），该通道存储 `Message` 对象列表，并用 reducer 函数对其进行注解（参见下方示例中的 `messages` 键）。Reducer 函数对于告诉图如何用每次状态更新（例如，当节点发送更新时）来更新状态中的 `Message` 对象列表至关重要。如果你不指定 reducer，每次状态更新都会用最近提供的值覆盖消息列表。如果你想简单地将消息附加到现有列表中，可以使用 `operator.add` 作为 reducer。

然而，你可能还想手动更新图状态中的消息（例如，人工干预）。如果你使用 `operator.add`，你发送到图的手动状态更新将被附加到现有消息列表中，而不是更新现有消息。为避免这种情况，你需要一个可以跟踪消息 ID 并覆盖现有消息（如果已更新）的 reducer。要实现这一点，你可以使用预先构建的 `add_messages` 函数。对于全新的消息，它只会附加到现有列表中，但它也能正确处理对现有消息的更新。
:::

:::js
在许多情况下，将先前的对话历史作为消息列表存储在图状态中会很有帮助。为此，我们可以向图状态添加一个键（通道），该通道存储 `Message` 对象列表，并用 reducer 函数对其进行注解（参见下方示例中的 `messages` 键）。Reducer 函数对于告诉图如何用每次状态更新（例如，当节点发送更新时）来更新状态中的 `Message` 对象列表至关重要。如果你不指定 reducer，每次状态更新都会用最近提供的值覆盖消息列表。如果你想简单地将消息附加到现有列表中，可以使用一个连接数组的函数作为 reducer。

然而，你可能还想手动更新图状态中的消息（例如，人工干预）。如果你使用简单的连接函数，你发送到图的手动状态更新将被附加到现有消息列表中，而不是更新现有消息。为避免这种情况，你需要一个可以跟踪消息 ID 并覆盖现有消息（如果已更新）的 reducer。要实现这一点，你可以使用预先构建的 `MessagesZodState` schema。对于全新的消息，它只会附加到现有列表中，但它也能正确处理对现有消息的更新。
:::

#### Serialization（序列化）

:::python
除了跟踪消息 ID，`add_messages` 函数在接收到 `messages` 通道的状态更新时，还会尝试将消息反序列化为 LangChain `Message` 对象。有关 LangChain 序列化/反序列化的更多信息，请参阅 [此处](https://python.langchain.com/docs/how_to/serialization/)。这允许使用以下格式发送图输入/状态更新：

```python
# 支持此格式
{"messages": [HumanMessage(content="message")]}

# 并且此格式也受支持
{"messages": [{"type": "human", "content": "message"}]}
```

由于在使用 `add_messages` 时，状态更新总是被反序列化为 LangChain `Messages`，因此你应该使用点符号访问消息属性，例如 `state["messages"][-1].content`。下面是一个使用 `add_messages` 作为其 reducer 函数的图的示例。

```python
from langchain_core.messages import AnyMessage
from langgraph.graph.message import add_messages
from typing import Annotated
from typing_extensions import TypedDict

class GraphState(TypedDict):
    messages: Annotated[list[AnyMessage], add_messages]
```

:::

:::js
除了跟踪消息 ID，`MessagesZodState` 在接收到 `messages` 通道的状态更新时，还会尝试将消息反序列化为 LangChain `Message` 对象。这允许使用以下格式发送图输入/状态更新：

```typescript
// 支持此格式
{
  messages: [new HumanMessage("message")];
}

// 并且此格式也受支持
{
  messages: [{ role: "human", content: "message" }];
}
```

由于在使用 `MessagesZodState` 时，状态更新总是被反序列化为 LangChain `Messages`，因此你应该使用点符号访问消息属性，例如 `state.messages[state.messages.length - 1].content`。下面是一个使用 `MessagesZodState` 的图的示例。

```typescript
import { StateGraph, MessagesZodState } from "@langchain/langgraph";

const graph = new StateGraph(MessagesZodState)
  ...
```

`MessagesZodState` 定义了一个单一的 `messages` 键，它是一个 `BaseMessage` 对象列表，并使用适当的 reducer。通常，需要跟踪的状态不仅仅是消息，所以我们会看到人们扩展此状态并添加更多字段，例如：

```typescript
const State = z.object({
  messages: MessagesZodState.shape.messages,
  documents: z.array(z.string()),
});
```

:::

:::python

#### MessagesState

由于在状态中包含消息列表非常普遍，因此我们预先构建了名为 `MessagesState` 的状态，它简化了消息的使用。`MessagesState` 定义了一个单一的 `messages` 键，该键是一个 `AnyMessage` 对象列表，并使用 `add_messages` reducer。通常，需要跟踪的状态不仅仅是消息，所以我们会看到人们子类化此状态并添加更多字段，例如：

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    documents: list[str]
```

:::

## Nodes（节点）

:::python

在 LangGraph 中，节点是 Python 函数（同步或异步），它们接受以下参数：

1. `state`: 图的 [state](#state)
2. `config`: 一个 `RunnableConfig` 对象，其中包含配置信息，如 `thread_id` 和 tracing 信息，如 `tags`
3. `runtime`: 一个 `Runtime` 对象，其中包含 [runtime `context`](#runtime-context) 以及其他信息，如 `store` 和 `stream_writer`

类似于 `NetworkX`，你可以使用 @[add_node][add_node] 方法将这些节点添加到图中：

```python
from dataclasses import dataclass
from typing_extensions import TypedDict

from langchain_core.runnables import RunnableConfig
from langgraph.graph import StateGraph
from langgraph.runtime import Runtime

class State(TypedDict):
    input: str
    results: str

@dataclass
class Context:
    user_id: str

builder = StateGraph(State)

def plain_node(state: State):
    return state

def node_with_runtime(state: State, runtime: Runtime[Context]):
    print("In node: ", runtime.context.user_id)
    return {"results": f"Hello, {state['input']}!"}

def node_with_config(state: State, config: RunnableConfig):
    print("In node with thread_id: ", config["configurable"]["thread_id"])
    return {"results": f"Hello, {state['input']}!"}


builder.add_node("plain_node", plain_node)
builder.add_node("node_with_runtime", node_with_runtime)
builder.add_node("node_with_config", node_with_config)
...
```

:::

:::js

在 LangGraph 中，节点通常是函数（同步或异步），它们接受以下参数：

1. `state`: 图的 [state](#state)
2. `config`: 一个 `RunnableConfig` 对象，其中包含配置信息，如 `thread_id` 和 tracing 信息，如 `tags`

你可以使用 `addNode` 方法将节点添加到图中。

```typescript
import { StateGraph } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

const State = z.object({
  input: z.string(),
  results: z.string(),
});

const builder = new StateGraph(State);
  .addNode("myNode", (state, config) => {
    console.log("In node: ", config?.configurable?.user_id);
    return { results: `Hello, ${state.input}!` };
  })
  addNode("otherNode", (state) => {
    return state;
  })
  ...
```

:::

后台会将函数转换为 [RunnableLambda](https://python.langchain.com/api_reference/core/runnables/langchain_core.runnables.base.RunnableLambda.html)，为你的函数添加批量和异步支持，以及原生的 tracing 和调试功能。

如果你向图添加一个节点而不指定名称，它将获得一个等同于函数名称的默认名称。

:::python

```python
builder.add_node(my_node)
# 然后你可以通过引用 "my_node" 来创建到该节点的边
```

:::

:::js

```typescript
builder.addNode(myNode);
// 然后你可以通过引用 "myNode" 来创建到该节点的边
```

:::

### `START` Node（`START` 节点）

`START` 节点是一个特殊节点，代表将用户输入发送到图的节点。引用此节点的主要目的是确定应该首先调用哪些节点。

:::python

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

:::

:::js

```typescript
import { START } from "@langchain/langgraph";

graph.addEdge(START, "nodeA");
```

:::

### `END` Node（`END` 节点）

`END` 节点是一个特殊节点，代表一个终止节点。当你希望表示操作完成后没有后续步骤的边时，会引用此节点。

:::python

```python
from langgraph.graph import END

graph.add_edge("node_a", END)
```

:::

:::js

```typescript
import { END } from "@langchain/langgraph";

graph.addEdge("nodeA", END);
```

:::

### Node Caching（节点缓存）

:::python
LangGraph 支持基于节点输入的任务/节点缓存。使用缓存：

- 在编译图（或指定入口点）时指定缓存（or specifying an entrypoint）
- 为节点指定缓存策略。每个缓存策略支持：
  - `key_func` 用于根据节点输入生成缓存键，默认为输入的 pickle 化的 `hash`。
  - `ttl`，缓存的生存时间（秒）。如果未指定，则缓存永远不会过期。

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
    # 昂贵的计算
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

1. 第一次运行花费两秒钟（由于模拟的昂贵计算）。
2. 第二次运行利用缓存并快速返回。
   :::

:::js
LangGraph 支持基于节点输入的任务/节点缓存。使用缓存：

- 在编译图（或指定入口点）时指定缓存（or specifying an entrypoint）
- 为节点指定缓存策略。每个缓存策略支持：
  - `keyFunc`，用于根据节点输入生成缓存键。
  - `ttl`，缓存的生存时间（秒）。如果未指定，则缓存永远不会过期。

```typescript
import { StateGraph, MessagesZodState } from "@langchain/langgraph";
import { InMemoryCache } from "@langchain/langgraph-checkpoint";

const graph = new StateGraph(MessagesZodState)
  .addNode(
    "expensive_node",
    async () => {
      // 模拟一个昂贵的操作
      await new Promise((resolve) => setTimeout(resolve, 3000));
      return { result: 10 };
    },
    { cachePolicy: { ttl: 3 } }
  )
  .addEdge(START, "expensive_node")
  .compile({ cache: new InMemoryCache() });

await graph.invoke({ x: 5 }, { streamMode: "updates" }); // (1)!
// [{"expensive_node": {"result": 10}}]
await graph.invoke({ x: 5 }, { streamMode: "updates" }); // (2)!
// [{"expensive_node": {"result": 10}, "__metadata__": {"cached": true}}]
```

:::

## Edges（边）

边定义了逻辑如何路由以及图如何决定停止。这是你的 agent 工作方式以及不同节点之间如何通信的重要组成部分。有几种关键的边类型：

- Normal Edges（普通边）：直接从一个节点到下一个节点。
- Conditional Edges（条件边）：调用一个函数来确定下一个要去的节点。
- Entry Point（入口点）：用户输入到达时首先调用哪个节点。
- Conditional Entry Point（条件入口点）：用户输入到达时调用一个函数来确定首先要调用的节点。

一个节点可以有 **多个** 出边。如果一个节点有多个出边，**所有** 这些目标节点将在下一个超级步中并行执行。

### Normal Edges（普通边）

:::python
如果你 **总是** 想从节点 A 转到节点 B，你可以直接使用 @[add_edge][add_edge] 方法。

```python
graph.add_edge("node_a", "node_b")
```

:::

:::js
如果你 **总是** 想从节点 A 转到节点 B，你可以直接使用 @[`addEdge`][add_edge] 方法。

```typescript
graph.addEdge("nodeA", "nodeB");
```

:::

### Conditional Edges（条件边）

:::python
如果你想 **可选地** 路由到一个或多个边（或可选地终止），你可以使用 @[add_conditional_edges][add_conditional_edges] 方法。此方法接受一个节点名称和一个在执行该节点后调用的“路由函数”：

```python
graph.add_conditional_edges("node_a", routing_function)
```

与节点类似，`routing_function` 接受图的当前 `state` 并返回一个值。

默认情况下，`routing_function` 的返回值用作下一个要将 state 发送到的节点（或节点列表）的名称。所有这些节点将在下一个超级步中并行运行。

你可以选择提供一个字典，将 `routing_function` 的输出映射到下一个节点的名称。

```python
graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
```

:::

:::js
如果你想 **可选地** 路由到一个或多个边（或可选地终止），你可以使用 @[`addConditionalEdges`][add_conditional_edges] 方法。此方法接受一个节点名称和一个在执行该节点后调用的“路由函数”：

```typescript
graph.addConditionalEdges("nodeA", routingFunction);
```

与节点类似，`routingFunction` 接受图的当前 `state` 并返回一个值。

默认情况下，`routingFunction` 的返回值用作下一个要将 state 发送到的节点（或节点列表）的名称。所有这些节点将在下一个超级步中并行运行。

你可以选择提供一个对象，将 `routingFunction` 的输出映射到下一个节点的名称。

```typescript
graph.addConditionalEdges("nodeA", routingFunction, {
  true: "nodeB",
  false: "nodeC",
});
```

:::

!!! tip

    如果你想在单个函数中结合状态更新和路由，请使用 [`Command`](#command) 而不是条件边。

### Entry Point（入口点）

:::python
入口点是图启动时运行的第一个节点（或节点集合）。你可以使用虚拟 @[`START`][START] 节点到第一个要执行的节点之间的 @[`add_edge`][add_edge] 方法来指定进入图的位置。

```python
from langgraph.graph import START

graph.add_edge(START, "node_a")
```

:::

:::js
入口点是图启动时运行的第一个节点（或节点集合）。你可以使用虚拟 @[`START`][START] 节点到第一个要执行的节点之间的 @[`addEdge`][add_edge] 方法来指定进入图的位置。

```typescript
import { START } from "@langchain/langgraph";

graph.addEdge(START, "nodeA");
```

:::

### Conditional Entry Point（条件入口点）

:::python
条件入口点允许你根据自定义逻辑从不同的节点开始。你可以使用虚拟 @[`START`][START] 节点到 @[`add_conditional_edges`][add_conditional_edges] 方法来实现这一点。

```python
from langgraph.graph import START

graph.add_conditional_edges(START, routing_function)
```

你可以选择提供一个字典，将 `routing_function` 的输出映射到下一个节点的名称。

```python
graph.add_conditional_edges(START, routing_function, {True: "node_b", False: "node_c"})
```

:::

:::js
条件入口点允许你根据自定义逻辑从不同的节点开始。你可以使用虚拟 @[`START`][START] 节点到 @[`addConditionalEdges`][add_conditional_edges] 方法来实现这一点。

```typescript
import { START } from "@langchain/langgraph";

graph.addConditionalEdges(START, routingFunction);
```

你可以选择提供一个对象，将 `routingFunction` 的输出映射到下一个节点的名称。

```typescript
graph.addConditionalEdges(START, routingFunction, {
  true: "nodeB",
  false: "nodeC",
});
```

:::

## `Send`

:::python
默认情况下，`Nodes` 和 `Edges` 都是预先定义的，并且操作于相同的共享状态。然而，在某些情况下，确切的边可能不是预先已知的，并且/或者你可能希望同时存在不同版本的 `State`。一个常见的例子是 [map-reduce](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 设计模式。在此设计模式中，第一个节点可能会生成一个对象列表，而你可能希望将其他节点应用于所有这些对象。对象的数量可能无法提前得知（意味着边的数量也可能未知），并且下游 `Node` 的输入 `State` 应该不同（每个生成对象一个）。

为了支持此设计模式，LangGraph 支持从条件边返回 @[`Send`][Send] 对象。`Send` 接受两个参数：第一个是节点名称，第二个是要传递给该节点的状态。

```python
def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state['subjects']]

graph.add_conditional_edges("node_a", continue_to_jokes)
```

:::

:::js
默认情况下，`Nodes` 和 `Edges` 都是预先定义的，并且操作于相同的共享状态。然而，在某些情况下，确切的边可能不是预先已知的，并且/或者你可能希望同时存在不同版本的 `State`。一个常见的例子是 map-reduce 设计模式。在此设计模式中，第一个节点可能会生成一个对象列表，而你可能希望将其他节点应用于所有这些对象。对象的数量可能无法提前得知（意味着边的数量也可能未知），并且下游 `Node` 的输入 `State` 应该不同（每个生成对象一个）。

为了支持此设计模式，LangGraph 支持从条件边返回 @[`Send`][Send] 对象。`Send` 接受两个参数：第一个是节点名称，第二个是要传递给该节点的状态。

```typescript
import { Send } from "@langchain/langgraph";

graph.addConditionalEdges("nodeA", (state) => {
  return state.subjects.map((subject) => new Send("generateJoke", { subject }));
});
```

:::

## `Command`

:::python
将控制流（边）和状态更新（节点）结合起来可能很有用。例如，你可能希望在 **同一节点** 中执行状态更新 **并** 决定下一个要去哪个节点。LangGraph 提供了一种方法，通过从节点函数返回 @[`Command`][Command] 对象来实现：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # 状态更新
        update={"foo": "bar"},
        # 控制流
        goto="my_other_node"
    )
```

使用 `Command`，你还可以实现动态控制流行为（与 [条件边](#conditional-edges) 相同）：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    if state["foo"] == "bar":
        return Command(update={"foo": "baz"}, goto="my_other_node")
```

:::

:::js
将控制流（边）和状态更新（节点）结合起来可能很有用。例如，你可能希望在 **同一节点** 中执行状态更新 **并** 决定下一个要去哪个节点。LangGraph 提供了一种方法，通过从节点函数返回 `Command` 对象来实现：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "myOtherNode",
  });
});
```

使用 `Command`，你还可以实现动态控制流行为（与 [条件边](#conditional-edges) 相同）：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  if (state.foo === "bar") {
    return new Command({
      update: { foo: "baz" },
      goto: "myOtherNode",
    });
  }
});
```

在节点函数中使用 `Command` 时，必须在添加节点时添加 `ends` 参数，以指定它可以路由到的节点：

```typescript
builder.addNode("myNode", myNode, {
  ends: ["myOtherNode", END],
});
```

:::

!!! important

    从节点函数返回 `Command` 时，必须添加返回类型注解，其中包含节点可以路由到的节点名称列表，例如 `Command[Literal["my_other_node"]]`。这对于图渲染是必需的，并且告诉 LangGraph `my_node` 可以导航到 `my_other_node`。

查看这个 [操作指南](../how-tos/graph-api.md#combine-control-flow-and-state-updates-with-command)，了解如何使用 `Command` 的完整端到端示例。

### When should I use Command instead of conditional edges?（何时使用 Command 而非条件边？）

- 当你需要 **同时** 更新图状态 **并** 路由到另一个节点时，使用 `Command`。例如，在实现 [多 agent 协作](./multi_agent.md#handoffs) 时，将状态传递给不同 agent 非常重要。
- 使用 [条件边](#conditional-edges) 在不更新状态的情况下有条件地路由节点之间。

### Navigating to a node in a parent graph（导航到父图中的节点）

:::python
如果你正在使用 [子图](./subgraphs.md)，你可能希望从子图中的节点导航到另一个子图（即父图中的另一个节点）。为此，可以在 `Command` 中指定 `graph=Command.PARENT`：

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

    当你从子图节点向父图节点发送共享了父子图 [state schemas](#schema) 的键的更新时，**必须** 为你在父图状态中更新的键定义一个 [reducer](#reducers)。请参阅此 [示例](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph)。

:::

:::js
如果你正在使用 [子图](./subgraphs.md)，你可能希望从子图中的节点导航到另一个子图（即父图中的另一个节点）。为此，可以在 `Command` 中指定 `graph: Command.PARENT`：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph", // 其中 `otherSubgraph` 是父图中的一个节点
    graph: Command.PARENT,
  });
});
```

!!! note

    将 `graph` 设置为 `Command.PARENT` 将导航到最近的父图。

!!! important "使用 `Command.PARENT` 进行状态更新"

    当你从子图节点向父图节点发送共享了父子图 [state schemas](#schema) 的键的更新时，**必须** 为你在父图状态中更新的键定义一个 [reducer](#reducers)。

:::

:::js
如果你正在使用 [子图](./subgraphs.md)，你可能希望从子图中的节点导航到另一个子图（即父图中的另一个节点）。为此，可以在 `Command` 中指定 `graph: Command.PARENT`：

```typescript
import { Command } from "@langchain/langgraph";

graph.addNode("myNode", (state) => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph", // 其中 `otherSubgraph` 是父图中的一个节点
    graph: Command.PARENT,
  });
});
```

!!! note

    将 `graph` 设置为 `Command.PARENT` 将导航到最近的父图。

!!! important "使用 `Command.PARENT` 进行状态更新"

    当你从子图节点向父图节点发送共享了父子图 [state schemas](#schema) 的键的更新时，**必须** 为你在父图状态中更新的键定义一个 [reducer](#reducers)。

:::

这在实现 [多 agent 协作](./multi_agent.md#handoffs) 时尤其有用。

查看 [此指南](../how-tos/graph-api.md#navigate-to-a-node-in-a-parent-graph) 以获取详细信息。

### Using inside tools（在工具中使用）

一个常见的用例是在工具内部更新图状态。例如，在客户支持应用程序中，你可能希望在对话开始时根据客户的账号或 ID 查找客户信息。

有关详细信息，请参阅 [此指南](../how-tos/graph-api.md#use-inside-tools)。

### Human-in-the-loop（人工干预）

:::python
`Command` 是人工干预工作流的重要组成部分：当使用 `interrupt()` 收集用户输入时，`Command` 用于提供输入并通过 `Command(resume="User input")` 恢复执行。有关更多信息，请参阅 [此概念指南](./human_in_the_loop.md)。
:::

:::js
`Command` 是人工干预工作流的重要组成部分：当使用 `interrupt()` 收集用户输入时，`Command` 用于提供输入并通过 `new Command({ resume: "User input" })` 恢复执行。有关更多信息，请参阅 [人工干预概念指南](./human_in_the_loop.md)。
:::

## Graph Migrations（图迁移）

LangGraph 可以轻松处理图定义（节点、边和状态）的迁移，即使在使用 checkpointer 跟踪状态时也是如此。

- 对于图末尾的线程（即未中断的），你可以更改图的整个拓扑结构（即所有节点和边，删除、添加、重命名等）。
- 对于当前中断的线程，我们支持除重命名/删除节点外的所有拓扑更改（因为该线程现在可能即将进入一个已不存在的节点）——如果这是一个阻塞项，请联系我们，我们可以优先处理解决方案。
- 对于修改状态，我们对添加和删除键具有完全的向后和向前兼容性。
- 重命名的状态键会在现有线程中丢失其保存的状态。
- 类型以不兼容方式更改的状态键可能会在从更改之前的状态的线程中导致问题——如果这是一个阻塞项，请联系我们，我们可以优先处理解决方案。

:::python

## Runtime Context（运行时上下文）

创建图时，可以为传递给节点的 `context_schema` 指定运行时上下文。这对于将不属于图状态的信息传递给节点很有用。例如，你可能想传递模型名称或数据库连接等依赖项。

```python
@dataclass
class ContextSchema:
    llm_provider: str = "openai"

graph = StateGraph(State, context_schema=ContextSchema)
```

:::

:::js

创建图时，你也可以标记图的某些部分是可配置的。这通常是为了能够轻松地在模型或系统提示之间切换。这允许你创建一个单一的“认知架构”（图），但拥有多个实例。

你可以在创建图时选择性地指定一个配置 schema。

```typescript
import { z } from "zod";

const ConfigSchema = z.object({
  llm: z.string(),
});

const graph = new StateGraph(State, ConfigSchema);
```

:::

:::python
然后，你可以通过 `invoke` 方法的 `context` 参数将此上下文传递到图中。

```python
graph.invoke(inputs, context={"llm_provider": "anthropic"})
```

:::

:::js
然后，你可以通过 `configurable` 配置字段将此配置传递到图中。

```typescript
const config = { configurable: { llm: "anthropic" } };

await graph.invoke(inputs, config);
```

:::

然后，你可以在节点或条件边内部访问和使用此上下文：

```python
from langgraph.runtime import Runtime

def node_a(state: State, runtime: Runtime[ContextSchema]):
    llm = get_llm(runtime.context.llm_provider)
    ...
```

有关配置的完整说明，请参阅 [此指南](../how-tos/graph-api.md#add-runtime-configuration)。
:::

:::js

```typescript
graph.addNode("myNode", (state, config) => {
  const llmType = config?.configurable?.llm || "openai";
  const llm = getLlm(llmType);
  return { results: `Hello, ${state.input}!` };
});
```

:::

### Recursion Limit（递归限制）

:::python
递归限制设置了图在单次执行期间可以执行的 [超级步](#graphs) 的最大数量。一旦达到限制，LangGraph 将引发 `GraphRecursionError`。默认情况下，此值为 25 步。递归限制可以在运行时设置在任何图上，并通过 config 字典传递给 `.invoke`/`.stream`。重要的是，`recursion_limit` 是一个独立的 `config` 键，不应像所有其他用户定义的配置一样传递到 `configurable` 键内。请参阅下面的示例：

```python
graph.invoke(inputs, config={"recursion_limit": 5}, context={"llm": "anthropic"})
```

阅读 [此操作指南](https://langchain-ai.github.io/langgraph/how-tos/recursion-limit/) 以了解递归限制的工作原理。
:::

:::js
递归限制设置了图在单次执行期间可以执行的 [超级步](#graphs) 的最大数量。一旦达到限制，LangGraph 将引发 `GraphRecursionError`。默认情况下，此值为 25 步。递归限制可以在运行时设置在任何图上，并通过 config 对象传递给 `.invoke`/`.stream`。重要的是，`recursionLimit` 是一个独立的 `config` 键，不应像所有其他用户定义的配置一样传递到 `configurable` 键内。请参阅下面的示例：

```typescript
await graph.invoke(inputs, {
  recursionLimit: 5,
  configurable: { llm: "anthropic" },
});
```

:::

## Visualization（可视化）

通常，能够可视化图很有用，尤其是在它们变得更加复杂时。LangGraph 提供了几种内置的可视化图的方法。有关更多信息，请参阅 [此操作指南](../how-tos/graph-api.md#visualize-your-graph)。