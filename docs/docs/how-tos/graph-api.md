# 如何使用 Graph API

本指南将演示 LangGraph 的 Graph API 基础知识。它将引导您了解 [state](#define-and-update-state) 的定义和更新，以及如何组合常见的图结构，例如 [sequences](#create-a-sequence-of-steps)、[branches](#create-branches) 和 [loops](#create-and-control-loops)。此外，它还将介绍 LangGraph 的控制功能，包括用于 map-reduce 工作流的 [Send API](#map-reduce-and-the-send-api) 和用于将状态更新与节点之间的“跳转”结合使用的 [Command API](#combine-control-flow-and-state-updates-with-command)。

## 设置

:::python
安装 `langgraph`:

```bash
pip install -U langgraph
```
:::

:::js
安装 `langgraph`:

```bash
npm install @langchain/langgraph
```
:::

!!! tip "设置 LangSmith 以获得更好的调试体验"

    注册 [LangSmith](https://smith.langchain.com) 可以快速发现问题并改进您的 LangGraph 项目性能。LangSmith 允许您使用跟踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用——在 [文档](https://docs.smith.langchain.com) 中可以找到更多关于如何开始的信息。

## 定义和更新状态

这里我们将展示如何在 LangGraph 中定义和更新 [state](../concepts/low_level.md#state)。我们将演示：

1. 如何使用 state 来定义图的 [schema](../concepts/low_level.md#schema)
2. 如何使用 [reducers](../concepts/low_level.md#reducers) 来控制 state 更新的处理方式。

### 定义状态

:::python
LangGraph 中的 [State](../concepts/low_level.md#state) 可以是 `TypedDict`、`Pydantic` 模型或 dataclass。下面我们将使用 `TypedDict`。有关使用 Pydantic 的详细信息，请参见 [本节](#use-pydantic-models-for-graph-state)。
:::

:::js
LangGraph 中的 [State](../concepts/low_level.md#state) 可以使用 Zod schema 来定义。下面我们将使用 Zod。有关使用其他方法的详细信息，请参见 [本节](#alternative-state-definitions)。
:::

默认情况下，图的输入和输出 schema 相同，并且 state 决定了该 schema。有关如何定义不同的输入和输出 schema 的信息，请参见 [本节](#define-input-and-output-schemas)。

让我们看一个使用 [messages](../concepts/low_level.md#working-with-messages-in-graph-state) 的简单示例。这代表了许多 LLM 应用中 state 的一种多功能表示方式。有关更多详细信息，请参见我们的 [concepts 页面](../concepts/low_level.md#working-with-messages-in-graph-state)。

:::python
```python
from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

此 state 包含一个 [message](https://python.langchain.com/docs/concepts/messages/) 对象列表，以及一个额外的整数字段。
:::

:::js
```typescript
import { BaseMessage } from "@langchain/core/messages";
import { z } from "zod";

const State = z.object({
  messages: z.array(z.custom<BaseMessage>()),
  extraField: z.number(),
});
```

此 state 包含一个 [message](https://js.langchain.com/docs/concepts/messages/) 对象列表，以及一个额外的整数字段。
:::

### 更新状态

:::python
让我们构建一个包含单个节点的示例图。我们的 [node](../concepts/low_level.md#nodes) 只是一个 Python 函数，它读取图的 state 并对其进行更新。此函数的第一个参数始终是 state：

```python
from langchain_core.messages import AIMessage

def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    return {"messages": messages + [new_message], "extra_field": 10}
```

此节点只是将一条消息附加到消息列表中，并填充一个额外字段。
:::

:::js
让我们构建一个包含单个节点的示例图。我们的 [node](../concepts/low_level.md#nodes) 只是一个 TypeScript 函数，它读取图的 state 并对其进行更新。此函数的第一个参数始终是 state：

```typescript
import { AIMessage } from "@langchain/core/messages";

const node = (state: z.infer<typeof State>) => {
  const messages = state.messages;
  const newMessage = new AIMessage("Hello!");
  return { messages: messages.concat([newMessage]), extraField: 10 };
};
```

此节点只是将一条消息附加到消息列表中，并填充一个额外字段。
:::

!!! important

    节点应直接返回 state 的更新，而不是修改 state。

:::python
接下来，让我们定义一个包含此节点的简单图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义一个在此 state 上运行的图。然后，我们使用 [add_node](../concepts/low_level.md#nodes) 来填充我们的图。

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node(node)
builder.set_entry_point("node")
graph = builder.compile()
```
:::

:::js
接下来，让我们定义一个包含此节点的简单图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义一个在此 state 上运行的图。然后，我们使用 [addNode](../concepts/low_level.md#nodes) 来填充我们的图。

```typescript
import { StateGraph } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("node", node)
  .addEdge("__start__", "node")
  .compile();
```
:::

LangGraph 提供了用于可视化图的内置实用程序。让我们检查一下我们的图。有关可视化的详细信息，请参见 [本节](#visualize-your-graph)。

:::python
```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![包含单个节点的简单图](assets/graph_api_image_1.png)
:::

:::js
```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

在本例中，我们的图只执行了一个节点。让我们继续一个简单的调用：

:::python
```python
from langchain_core.messages import HumanMessage

result = graph.invoke({"messages": [HumanMessage("Hi")]})
result
```

```
{'messages': [HumanMessage(content='Hi'), AIMessage(content='Hello!')], 'extra_field': 10}
```
:::

:::js
```typescript
import { HumanMessage } from "@langchain/core/messages";

const result = await graph.invoke({ messages: [new HumanMessage("Hi")], extraField: 0 });
console.log(result);
```

```
{ messages: [HumanMessage { content: 'Hi' }, AIMessage { content: 'Hello!' }], extraField: 10 }
```
:::

请注意：

- 我们通过更新 state 的一个键来启动调用。
- 我们在调用结果中收到了整个 state。

:::python
为了方便起见，我们经常通过美观打印来检查 [message 对象](https://python.langchain.com/docs/concepts/messages/) 的内容：

```python
for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```
:::

:::js
为了方便起见，我们经常通过日志记录来检查 [message 对象](https://js.langchain.com/docs/concepts/messages/) 的内容：

```typescript
for (const message of result.messages) {
  console.log(`${message.getType()}: ${message.content}`);
}
```

```
human: Hi
ai: Hello!
```
:::

### 使用 reducers 处理状态更新

State 的每个键都可以有自己的独立 [reducer](../concepts/low_level.md#reducers) 函数，该函数控制如何应用来自节点的更新。如果未明确指定 reducer 函数，则假定对该键的所有更新都应覆盖其现有值。

:::python
对于 `TypedDict` 状态 schema，我们可以通过用 reducer 函数注解 state 的相应字段来定义 reducers。

在前面的示例中，我们的节点通过将消息附加到 `"messages"` 键来更新它。下面，我们向此键添加一个 reducer，以便自动附加更新：

```python
from typing_extensions import Annotated

def add(left, right):
    """也可以从 `operator` 内置模块导入 `add`。"""
    return left + right

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add]
    extra_field: int
```

现在我们的节点可以简化了：

```python
def node(state: State):
    new_message = AIMessage("Hello!")
    # highlight-next-line
    return {"messages": [new_message], "extra_field": 10}
```
:::

:::js
对于 Zod 状态 schema，我们可以通过使用特殊 `.langgraph.reducer()` 方法在 schema 字段上定义 reducers。

在前面的示例中，我们的节点通过将消息附加到 `"messages"` 键来更新它。下面，我们向此键添加一个 reducer，以便自动附加更新：

```typescript
import "@langchain/langgraph/zod";

const State = z.object({
  // highlight-next-line
  messages: z.array(z.custom<BaseMessage>()).langgraph.reducer((x, y) => x.concat(y)),
  extraField: z.number(),
});
```

现在我们的节点可以简化了：

```typescript
const node = (state: z.infer<typeof State>) => {
  const newMessage = new AIMessage("Hello!");
  // highlight-next-line
  return { messages: [newMessage], extraField: 10 };
};
```
:::

:::python
```python
from langgraph.graph import START

graph = StateGraph(State).add_node(node).add_edge(START, "node").compile()

result = graph.invoke({"messages": [HumanMessage("Hi")]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```
:::

:::js
```typescript
import { START } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("node", node)
  .addEdge(START, "node")
  .compile();

const result = await graph.invoke({ messages: [new HumanMessage("Hi")] });

for (const message of result.messages) {
  console.log(`${message.getType()}: ${message.content}`);
}
```

```
human: Hi
ai: Hello!
```
:::

#### MessagesState

实际上，更新消息列表需要考虑更多因素：

- 我们可能希望更新 state 中的现有消息。
- 我们可能希望接受 [message format](../concepts/low_level.md#using-messages-in-your-graph) 的简写，例如 [OpenAI format](https://python.langchain.com/docs/concepts/messages/#openai-format)。

:::python
LangGraph 包含一个内置的 reducer `add_messages`，它处理了这些注意事项：

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add_messages]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}

graph = StateGraph(State).add_node(node).set_entry_point("node").compile()
```

```python
# highlight-next-line
input_message = {"role": "user", "content": "Hi"}

result = graph.invoke({"messages": [input_message]})

for message in result["messages"]:
    message.pretty_print()
```

```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

这对于涉及 [chat models](https://python.langchain.com/docs/concepts/chat_models/) 的应用程序来说是一种多功能的状态表示。LangGraph 包含一个预置的 `MessagesState` 以方便使用，因此我们可以拥有：

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```
:::

:::js
LangGraph 包含一个内置的 `MessagesZodState`，它处理了这些注意事项：

```typescript
import { MessagesZodState } from "@langchain/langgraph";

const State = z.object({
  // highlight-next-line
  messages: MessagesZodState.shape.messages,
  extraField: z.number(),
});

const graph = new StateGraph(State)
  .addNode("node", (state) => {
    const newMessage = new AIMessage("Hello!");
    return { messages: [newMessage], extraField: 10 };
  })
  .addEdge(START, "node")
  .compile();
```

```typescript
// highlight-next-line
const inputMessage = { role: "user", content: "Hi" };

const result = await graph.invoke({ messages: [inputMessage] });

for (const message of result.messages) {
  console.log(`${message.getType()}: ${message.content}`);
}
```

```
human: Hi
ai: Hello!
```

这对于涉及 [chat models](https://js.langchain.com/docs/concepts/chat_models/) 的应用程序来说是一种多功能的状态表示。LangGraph 包含此预置的 `MessagesZodState` 以方便使用，因此我们可以拥有：

```typescript
import { MessagesZodState } from "@langchain/langgraph";

const State = MessagesZodState.extend({
  extraField: z.number(),
});
```
:::

### 定义输入和输出 schema

默认情况下，`StateGraph` 使用单个 schema 进行操作，并且所有节点都应使用该 schema 进行通信。但是，也可以为图定义单独的输入和输出 schema。

指定不同的 schema 时，仍将使用内部 schema 在节点之间进行通信。输入 schema 确保提供的输入与预期的结构匹配，而输出 schema 则会过滤内部数据，仅返回与定义的输出 schema 相符的相关信息。

下面我们将介绍如何定义不同的输入和输出 schema。

:::python
```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 输入的 schema 定义
class InputState(TypedDict):
    question: str

# 输出的 schema 定义
class OutputState(TypedDict):
    answer: str

# 整体 schema 定义，结合输入和输出
class OverallState(InputState, OutputState):
    pass

# 处理输入并生成答案的节点定义
def answer_node(state: InputState):
    # 示例答案和额外的键
    return {"answer": "bye", "question": state["question"]}

# 构建图，并指定输入和输出 schema
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)  # 添加 answer 节点
builder.add_edge(START, "answer_node")  # 定义起始边
builder.add_edge("answer_node", END)  # 定义结束边
graph = builder.compile()  # 编译图

# 使用输入调用图并打印结果
print(graph.invoke({"question": "hi"}))
```

```
{'answer': 'bye'}
```
:::

:::js
```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

// 输入的 schema 定义
const InputState = z.object({
  question: z.string(),
});

// 输出的 schema 定义
const OutputState = z.object({
  answer: z.string(),
});

// 整体 schema 定义，结合输入和输出
const OverallState = InputState.merge(OutputState);

// 构建图，并指定输入和输出 schema
const graph = new StateGraph({
  input: InputState,
  output: OutputState,
  state: OverallState,
})
  .addNode("answerNode", (state) => {
    // 示例答案和额外的键
    return { answer: "bye", question: state.question };
  })
  .addEdge(START, "answerNode")
  .addEdge("answerNode", END)
  .compile();

// 使用输入调用图并打印结果
console.log(await graph.invoke({ question: "hi" }));
```

```
{ answer: 'bye' }
```
:::

请注意，invoke 的输出仅包含 output schema。

### 传递私有状态

在某些情况下，您可能希望节点能够交换对于中间逻辑至关重要但不需要包含在图主 schema 中的信息。此私有数据与图的整体输入/输出无关，并且只能在特定节点之间共享。

下面我们将创建一个由三个节点（node_1、node_2 和 node_3）组成的简单顺序图，其中私有数据在第一个（node_1 和 node_2）和第二个（node_1 和 node_2）步骤之间传递，而第三个步骤（node_3）只能访问公共的整体 state。

:::python
```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 图的整体 state（这是跨节点共享的公共 state）
class OverallState(TypedDict):
    a: str

# node_1 的输出包含不属于整体 state 的私有数据
class Node1Output(TypedDict):
    private_data: str

# 私有数据仅在 node_1 和 node_2 之间共享
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "set by node_1"}
    print(f"Entered node `node_1`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# node_2 的输入仅请求 node_1 之后可用的私有数据
class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    output = {"a": "set by node_2"}
    print(f"Entered node `node_2`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# node_3 只能访问整体 state（无法访问 node_1 的私有数据）
def node_3(state: OverallState) -> OverallState:
    output = {"a": "set by node_3"}
    print(f"Entered node `node_3`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# 按顺序连接节点
# node_2 接受来自 node_1 的私有数据，而
# node_3 则看不到私有数据。
builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()

# 使用初始 state 调用图
response = graph.invoke(
    {
        "a": "set at start",
    }
)

print()
print(f"Output of graph invocation: {response}")
```

```
Entered node `node_1`:
	Input: {'a': 'set at start'}.
	Returned: {'private_data': 'set by node_1'}
Entered node `node_2`:
	Input: {'private_data': 'set by node_1'}.
	Returned: {'a': 'set by node_2'}
Entered node `node_3`:
	Input: {'a': 'set by node_2'}.
	Returned: {'a': 'set by node_3'}

Output of graph invocation: {'a': 'set by node_3'}
```
:::

:::js
```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

// 图的整体 state（这是跨节点共享的公共 state）
const OverallState = z.object({
  a: z.string(),
});

// node1 的输出包含不属于整体 state 的私有数据
const Node1Output = z.object({
  privateData: z.string(),
});

// 私有数据仅在 node1 和 node2 之间共享
const node1 = (state: z.infer<typeof OverallState>): z.infer<typeof Node1Output> => {
  const output = { privateData: "set by node1" };
  console.log(`Entered node 'node1':\n\tInput: ${JSON.stringify(state)}.\n\tReturned: ${JSON.stringify(output)}`);
  return output;
};

// node2 的输入仅请求 node1 之后可用的私有数据
const Node2Input = z.object({
  privateData: z.string(),
});

const node2 = (state: z.infer<typeof Node2Input>): z.infer<typeof OverallState> => {
  const output = { a: "set by node2" };
  console.log(`Entered node 'node2':\n\tInput: ${JSON.stringify(state)}.\n\tReturned: ${JSON.stringify(output)}`);
  return output;
};

// node3 只能访问整体 state（无法访问 node1 的私有数据）
const node3 = (state: z.infer<typeof OverallState>): z.infer<typeof OverallState> => {
  const output = { a: "set by node3" };
  console.log(`Entered node 'node3':\n\tInput: ${JSON.stringify(state)}.\n\tReturned: ${JSON.stringify(output)}`);
  return output;
};

// 按顺序连接节点
// node2 接受来自 node1 的私有数据，而
// node3 则看不到私有数据。
const graph = new StateGraph({
  state: OverallState,
  nodes: {
    node1: { action: node1, output: Node1Output },
    node2: { action: node2, input: Node2Input },
    node3: { action: node3 },
  }
})
  .addEdge(START, "node1")
  .addEdge("node1", "node2")
  .addEdge("node2", "node3")
  .compile();

// 使用初始 state 调用图
const response = await graph.invoke({ a: "set at start" });

console.log(`\nOutput of graph invocation: ${JSON.stringify(response)}`);
```

```
Entered node 'node1':
	Input: {"a":"set at start"}.
	Returned: {"privateData":"set by node1"}
Entered node 'node2':
	Input: {"privateData":"set by node1"}.
	Returned: {"a":"set by node2"}
Entered node 'node3':
	Input: {"a":"set by node2"}.
	Returned: {"a":"set by node3"}

Output of graph invocation: {"a":"set by node3"}
```
:::

:::python

### 使用 Pydantic 模型定义图状态

[StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph) 在初始化时接受一个 `state_schema` 参数，该参数指定了图中节点可以访问和更新的 state 的“形状”。

在我们的示例中，我们通常使用 Python 内置的 `TypedDict` 或 [`dataclass`](https://docs.python.org/3/library/dataclasses.html) 来设置 `state_schema`，但 `state_schema` 可以是任何 [类型](https://docs.python.org/3/library/stdtypes.html#type-objects)。

这里，我们将演示如何使用 [Pydantic BaseModel](https://docs.pydantic.dev/latest/api/base_model/) 作为 `state_schema`，为**输入**添加运行时验证。

!!! note "已知限制"

    - 目前，图的输出**不会**是 pydantic 模型的实例。
    - 运行时验证仅发生在节点输入时，而不发生在节点输出时。
    - Pydantic 的验证错误跟踪无法显示错误出现在哪个节点。
    - Pydantic 的递归验证可能很慢。对于性能敏感的应用，您可能需要考虑使用 `dataclass`。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from pydantic import BaseModel

# 图的整体 state（这是跨节点共享的公共 state）
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}

# 构建状态图
builder = StateGraph(OverallState)
builder.add_node(node)  # node_1 是第一个节点
builder.add_edge(START, "node")  # 使用 node_1 启动图
builder.add_edge("node", END)  # 在 node_1 之后结束图
graph = builder.compile()

# 使用有效输入测试图
graph.invoke({"a": "hello"})
```

使用**无效**输入调用图

```python
try:
    graph.invoke({"a": 123})  # 应该是字符串
except Exception as e:
    print("由于 `a` 是整数而不是字符串，因此引发了异常。")
    print(e)
```

```
由于 `a` 是整数而不是字符串，因此引发了异常。
1 validation error for OverallState
a
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
    For further information visit https://errors.pydantic.dev/2.9/v/string_type
```

有关 Pydantic 模型状态的其他功能的更多信息，请参阅下文：

??? example "序列化行为"

    当使用 Pydantic 模型作为状态 schema 时，理解序列化工作方式非常重要，尤其是在以下情况：
    - 将 Pydantic 对象作为输入进行传递
    - 接收图的输出
    - 处理嵌套的 Pydantic 模型

    让我们通过实际示例了解这些行为。

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class NestedModel(BaseModel):
        value: str

    class ComplexState(BaseModel):
        text: str
        count: int
        nested: NestedModel

    def process_node(state: ComplexState):
        # 节点接收已验证的 Pydantic 对象
        print(f"Input state type: {type(state)}")
        print(f"Nested type: {type(state.nested)}")
        # 返回字典更新
        return {"text": state.text + " processed", "count": state.count + 1}

    # 构建图
    builder = StateGraph(ComplexState)
    builder.add_node("process", process_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)
    graph = builder.compile()

    # 创建 Pydantic 实例作为输入
    input_state = ComplexState(text="hello", count=0, nested=NestedModel(value="test"))
    print(f"Input object type: {type(input_state)}")

    # 使用 Pydantic 实例调用图
    result = graph.invoke(input_state)
    print(f"Output type: {type(result)}")
    print(f"Output content: {result}")

    # 如果需要，转换回 Pydantic 模型
    output_model = ComplexState(**result)
    print(f"Converted back to Pydantic: {type(output_model)}")
    ```

??? example "运行时类型强制转换"

    Pydantic 会对某些数据类型执行运行时类型强制转换。这可能很有用，但如果您不了解它，也可能导致意外行为。

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class CoercionExample(BaseModel):
        # Pydantic 会将字符串数字强制转换为整数
        number: int
        # Pydantic 会将字符串布尔值解析为 bool
        flag: bool

    def inspect_node(state: CoercionExample):
        print(f"number: {state.number} (type: {type(state.number)})")
        print(f"flag: {state.flag} (type: {type(state.flag)})")
        return {}

    builder = StateGraph(CoercionExample)
    builder.add_node("inspect", inspect_node)
    builder.add_edge(START, "inspect")
    builder.add_edge("inspect", END)
    graph = builder.compile()

    # 使用将被转换的字符串输入演示强制转换
    result = graph.invoke({"number": "42", "flag": "true"})

    # 这会因验证错误而失败
    try:
        graph.invoke({"number": "not-a-number", "flag": "true"})
    except Exception as e:
        print(f"\nExpected validation error: {e}")
    ```

??? example "使用消息模型"

    在状态 schema 中使用 LangChain 消息类型时，序列化存在重要注意事项。当通过网络传输消息对象时，您应该使用 `AnyMessage`（而不是 `BaseMessage`）以实现正确序列化/反序列化。

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel
    from langchain_core.messages import HumanMessage, AIMessage, AnyMessage
    from typing import List

    class ChatState(BaseModel):
        messages: List[AnyMessage]
        context: str

    def add_message(state: ChatState):
        return {"messages": state.messages + [AIMessage(content="Hello there!")]}

    builder = StateGraph(ChatState)
    builder.add_node("add_message", add_message)
    builder.add_edge(START, "add_message")
    builder.add_edge("add_message", END)
    graph = builder.compile()

    # 创建包含消息的输入
    initial_state = ChatState(
        messages=[HumanMessage(content="Hi")], context="Customer support chat"
    )

    result = graph.invoke(initial_state)
    print(f"Output: {result}")

    # 转换回 Pydantic 模型以查看消息类型
    output_model = ChatState(**result)
    for i, msg in enumerate(output_model.messages):
        print(f"Message {i}: {type(msg).__name__} - {msg.content}")
    ```
:::

:::js
### 其他状态定义

虽然 Zod schema 是推荐的方法，但 LangGraph 也支持其他定义状态 schema 的方式：

```typescript
import { BaseMessage } from "@langchain/core/messages";
import { StateGraph } from "@langchain/langgraph";

interface WorkflowChannelsState {
  messages: BaseMessage[];
  question: string;
  answer: string;
}

const workflowWithChannels = new StateGraph<WorkflowChannelsState>({
  channels: {
    messages: {
      reducer: (currentState, updateValue) => currentState.concat(updateValue),
      default: () => [],
    },
    question: null,
    answer: null,
  },
});
```
:::

## 添加运行时配置

有时您希望在调用图时配置它。例如，您可能希望能够在运行时指定要使用的 LLM 或系统提示，而**不要用这些参数污染图状态**。

要添加运行时配置：

1. 指定配置的 schema
2. 将配置添加到节点或条件边的函数签名中
3. 将配置传递给图。

下面是一个简单示例：

:::python
```python
from langgraph.graph import END, StateGraph, START
from langgraph.runtime import Runtime
from typing_extensions import TypedDict

# 1. 指定配置 schema
class ContextSchema(TypedDict):
    my_runtime_value: str

# 2. 定义一个在节点中访问配置的图
class State(TypedDict):
    my_state_value: str

# highlight-next-line
def node(state: State, runtime: Runtime[ContextSchema]):
    # highlight-next-line
    if runtime.context["my_runtime_value"] == "a":
        return {"my_state_value": 1}
        # highlight-next-line
    elif runtime.context["my_runtime_value"] == "b":
        return {"my_state_value": 2}
    else:
        raise ValueError("Unknown values.")

# highlight-next-line
builder = StateGraph(State, context_schema=ContextSchema)
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)

graph = builder.compile()

# 3. 在运行时传入配置：
# highlight-next-line
print(graph.invoke({}, context={"my_runtime_value": "a"}))
# highlight-next-line
print(graph.invoke({}, context={"my_runtime_value": "b"}))
```

```
{'my_state_value': 1}
{'my_state_value': 2}
```
:::

:::js
```typescript
import { StateGraph, END, START } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

// 1. 指定配置 schema
const ConfigurableSchema = z.object({
  myRuntimeValue: z.string(),
});

// 2. 定义一个在节点中访问配置的图
const State = z.object({
  myStateValue: z.number(),
});

const graph = new StateGraph(State)
  .addNode("node", (state, config) => {
    // highlight-next-line
    if (config?.configurable?.myRuntimeValue === "a") {
      return { myStateValue: 1 };
      // highlight-next-line
    } else if (config?.configurable?.myRuntimeValue === "b") {
      return { myStateValue: 2 };
    } else {
      throw new Error("Unknown values.");
    }
  })
  .addEdge(START, "node")
  .addEdge("node", END)
  .compile();

// 3. 在运行时传入配置：
// highlight-next-line
console.log(await graph.invoke({}, { configurable: { myRuntimeValue: "a" } }));
// highlight-next-line
console.log(await graph.invoke({}, { configurable: { myRuntimeValue: "b" } }));
```

```
{ myStateValue: 1 }
{ myStateValue: 2 }
```
:::

??? example "扩展示例：在运行时指定 LLM"

    :::python
    下面我们将演示一个在运行时配置要使用的 LLM 的实际示例。我们将使用 OpenAI 和 Anthropic 模型。

    ```python
    from dataclasses import dataclass

    from langchain.chat_models import init_chat_model
    from langgraph.graph import MessagesState, END, StateGraph, START
    from langgraph.runtime import Runtime
    from typing_extensions import TypedDict

    @dataclass
    class ContextSchema:
        model_provider: str = "anthropic"

    MODELS = {
        "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
        "openai": init_chat_model("openai:gpt-4.1-mini"),
    }

    def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
        model = MODELS[runtime.context.model_provider]
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    builder = StateGraph(MessagesState, context_schema=ContextSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # 用法
    input_message = {"role": "user", "content": "hi"}
    # 无配置，使用默认值（Anthropic）
    response_1 = graph.invoke({"messages": [input_message]}, context=ContextSchema())["messages"][-1]
    # 或者，可以设置为 OpenAI
    response_2 = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai"})["messages"][-1]

    print(response_1.response_metadata["model_name"])
    print(response_2.response_metadata["model_name"])
    ```
    ```
    claude-3-5-haiku-20241022
    gpt-4.1-mini-2025-04-14
    ```
    :::

    :::js
    下面我们将演示一个在运行时配置要使用的 LLM 的实际示例。我们将使用 OpenAI 和 Anthropic 模型。

    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { MessagesZodState, StateGraph, START, END } from "@langchain/langgraph";
    import { RunnableConfig } from "@langchain/core/runnables";
    import { z } from "zod";

    const ConfigSchema = z.object({
      modelProvider: z.string().default("anthropic"),
    });

    const MODELS = {
      anthropic: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
      openai: new ChatOpenAI({ model: "gpt-4o-mini" }),
    };

    const graph = new StateGraph(MessagesZodState)
      .addNode("model", async (state, config) => {
        const modelProvider = config?.configurable?.modelProvider || "anthropic";
        const model = MODELS[modelProvider as keyof typeof MODELS];
        const response = await model.invoke(state.messages);
        return { messages: [response] };
      })
      .addEdge(START, "model")
      .addEdge("model", END)
      .compile();

    // 用法
    const inputMessage = { role: "user", content: "hi" };
    // 无配置，使用默认值（Anthropic）
    const response1 = await graph.invoke({ messages: [inputMessage] });
    // 或者，可以设置为 OpenAI
    const response2 = await graph.invoke(
      { messages: [inputMessage] },
      { configurable: { modelProvider: "openai" } }
    );

    console.log(response1.messages.at(-1)?.response_metadata?.model);
    console.log(response2.messages.at(-1)?.response_metadata?.model);
    ```
    ```
    claude-3-5-haiku-20241022
    gpt-4o-mini-2024-07-18
    ```
    :::

??? example "扩展示例：运行时指定模型和系统消息"

    :::python
    下面我们将演示一个在运行时配置两个参数：LLM 和系统消息的实际示例。

    ```python
    from dataclasses import dataclass
    from typing import Optional
    from langchain.chat_models import init_chat_model
    from langchain_core.messages import SystemMessage
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.runtime import Runtime
    from typing_extensions import TypedDict

    @dataclass
    class ContextSchema:
        model_provider: str = "anthropic"
        system_message: str | None = None

    MODELS = {
        "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
        "openai": init_chat_model("openai:gpt-4.1-mini"),
    }

    def call_model(state: MessagesState, runtime: Runtime[ContextSchema]):
        model = MODELS[runtime.context.model_provider]
        messages = state["messages"]
        if (system_message := runtime.context.system_message):
            messages = [SystemMessage(system_message)] + messages
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState, context_schema=ContextSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # 用法
    input_message = {"role": "user", "content": "hi"}
    response = graph.invoke({"messages": [input_message]}, context={"model_provider": "openai", "system_message": "Respond in Italian."})
    for message in response["messages"]:
        message.pretty_print()
    ```
    :::

    :::js
    下面我们将演示一个在运行时配置两个参数：LLM 和系统消息的实际示例。

    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { SystemMessage } from "@langchain/core/messages";
    import { MessagesZodState, StateGraph, START, END } from "@langchain/langgraph";
    import { z } from "zod";

    const ConfigSchema = z.object({
      modelProvider: z.string().default("anthropic"),
      systemMessage: z.string().optional(),
    });

    const MODELS = {
      anthropic: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
      openai: new ChatOpenAI({ model: "gpt-4o-mini" }),
    };

    const graph = new StateGraph(MessagesZodState)
      .addNode("model", async (state, config) => {
        const modelProvider = config?.configurable?.modelProvider || "anthropic";
        const systemMessage = config?.configurable?.systemMessage;
        
        const model = MODELS[modelProvider as keyof typeof MODELS];
        let messages = state.messages;
        
        if (systemMessage) {
          messages = [new SystemMessage(systemMessage), ...messages];
        }
        
        const response = await model.invoke(messages);
        return { messages: [response] };
      })
      .addEdge(START, "model")
      .addEdge("model", END)
      .compile();

    // 用法
    const inputMessage = { role: "user", content: "hi" };
    const response = await graph.invoke(
      { messages: [inputMessage] },
      {
        configurable: {
          modelProvider: "openai",
          systemMessage: "Respond in Italian."
        }
      }
    );
    
    for (const message of response.messages) {
      console.log(`${message.getType()}: ${message.content}`);
    }
    ```
    ```
    human: hi
    ai: Ciao! Come posso aiutarti oggi?
    ```
    :::

## 添加重试策略

在许多情况下，您可能希望节点具有自定义的重试策略，例如在调用 API、查询数据库或调用 LLM 时。LangGraph 允许您为节点添加重试策略。

:::python
要配置重试策略，请将 `retry_policy` 参数传递给 [add_node](../reference/graphs.md#langgraph.graph.state.StateGraph.add_node) 参数。`retry_policy` 参数接受一个 `RetryPolicy` 命名元组对象。下面我们使用默认参数实例化一个 `RetryPolicy` 对象，并将其与一个节点关联：

```python
from langgraph.types import RetryPolicy

builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(),
)
```

默认情况下，`retry_on` 参数使用 `default_retry_on` 函数，该函数会重试除以下异常之外的所有异常：

- `ValueError`
- `TypeError`
- `ArithmeticError`
- `ImportError`
- `LookupError`
- `NameError`
- `SyntaxError`
- `RuntimeError`
- `ReferenceError`
- `StopIteration`
- `StopAsyncIteration`
- `OSError`

此外，对于来自 `requests` 和 `httpx` 等流行 HTTP 请求库的异常，它仅在 5xx 状态码时重试。
:::

:::js
要配置重试策略，请将 `retryPolicy` 参数传递给 [addNode](../reference/graphs.md#langgraph.graph.state.StateGraph.add_node) 参数。`retryPolicy` 参数接受一个 `RetryPolicy` 对象。下面我们使用默认参数实例化一个 `RetryPolicy` 对象，并将其与一个节点关联：

```typescript
import { RetryPolicy } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("nodeName", nodeFunction, { retryPolicy: {} })
  .compile();
```

默认情况下，重试策略会重试除以下异常之外的所有异常：

- `TypeError`
- `SyntaxError`
- `ReferenceError`
:::

??? example "扩展示例：自定义重试策略"

    :::python
    考虑一个从 SQL 数据库读取数据的示例。下面我们向节点传递两个不同的重试策略：

    ```python
    import sqlite3
    from typing_extensions import TypedDict
    from langchain.chat_models import init_chat_model
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.types import RetryPolicy
    from langchain_community.utilities import SQLDatabase
    from langchain_core.messages import AIMessage

    db = SQLDatabase.from_uri("sqlite:///:memory:")
    model = init_chat_model("anthropic:claude-3-5-haiku-latest")

    def query_database(state: MessagesState):
        query_result = db.run("SELECT * FROM Artist LIMIT 10;")
        return {"messages": [AIMessage(content=query_result)]}

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    # 定义一个新图
    builder = StateGraph(MessagesState)
    builder.add_node(
        "query_database",
        query_database,
        retry_policy=RetryPolicy(retry_on=sqlite3.OperationalError),
    )
    builder.add_node("model", call_model, retry_policy=RetryPolicy(max_attempts=5))
    builder.add_edge(START, "model")
    builder.add_edge("model", "query_database")
    builder.add_edge("query_database", END)
    graph = builder.compile()
    ```
    :::

    :::js
    考虑一个从 SQL 数据库读取数据的示例。下面我们向节点传递两个不同的重试策略：

    ```typescript
    import Database from "better-sqlite3";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, END, MessagesZodState } from "@langchain/langgraph";
    import { AIMessage } from "@langchain/core/messages";
    import { z } from "zod";

    // 创建一个内存数据库
    const db: typeof Database.prototype = new Database(":memory:");

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" });

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const response = await model.invoke(state.messages);
      return { messages: [response] };
    };

    const queryDatabase = async (state: z.infer<typeof MessagesZodState>) => {
      const queryResult: string = JSON.stringify(
        db.prepare("SELECT * FROM Artist LIMIT 10;").all(),
      );

      return { messages: [new AIMessage({ content: "queryResult" })] };
    };

    const workflow = new StateGraph(MessagesZodState)
      // 定义我们将循环的两个节点
      .addNode("call_model", callModel, { retryPolicy: { maxAttempts: 5 } })
      .addNode("query_database", queryDatabase, {
        retryPolicy: {
          retryOn: (e: any): boolean => {
            if (e instanceof Database.SqliteError) {
              // 重试 "SQLITE_BUSY" 错误
              return e.code === "SQLITE_BUSY";
            }
            return false; // 其他错误不重试
          },
        },
      })
      .addEdge(START, "call_model")
      .addEdge("call_model", "query_database")
      .addEdge("query_database", END);

    const graph = workflow.compile();
    ```
    :::

:::python

## 添加节点缓存

节点缓存适用于您希望避免重复操作的情况，例如在执行耗时（在时间和成本方面）的操作时。LangGraph 允许您为图中的节点添加单独的缓存策略。

要配置缓存策略，请将 `cache_policy` 参数传递给 [add_node](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.state.StateGraph.add_node) 函数。在下面的示例中，一个 [`CachePolicy`](https://langchain-ai.github.io/langgraph/reference/types/?h=cachepolicy#langgraph.types.CachePolicy) 对象被实例化，其生存时间（ttl）为 120 秒，并使用默认的 `key_func` 生成器。然后将其与一个节点关联：

```python
from langgraph.types import CachePolicy

builder.add_node(
    "node_name",
    node_function,
    cache_policy=CachePolicy(ttl=120),
)
```

然后，要为图启用节点级缓存，请在编译图时设置 `cache` 参数。下面的示例使用 `InMemoryCache` 设置一个带有内存缓存的图，但 `SqliteCache` 也可用。

```python
from langgraph.cache.memory import InMemoryCache

graph = builder.compile(cache=InMemoryCache())
```
:::

## 创建步骤序列

!!! info "先决条件"

    本指南假定您已熟悉关于 [state](#define-and-update-state) 的上一节。

这里我们将演示如何构建一个简单的步骤序列。我们将展示：

1. 如何构建顺序图
2. 用于构建类似图的内置简写。

:::python
要添加节点序列，我们使用 [graph](../concepts/low_level.md#stategraph) 的 `.add_node` 和 `.add_edge` 方法：

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# 添加节点
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# 添加边
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```

我们也可以使用内置的简写 `.add_sequence`：

```python
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
```
:::

:::js
要添加节点序列，我们使用 [graph](../concepts/low_level.md#stategraph) 的 `.addNode` 和 `.addEdge` 方法：

```typescript
import { START, StateGraph } from "@langchain/langgraph";

const builder = new StateGraph(State)
  .addNode("step1", step1)
  .addNode("step2", step2)
  .addNode("step3", step3)
  .addEdge(START, "step1")
  .addEdge("step1", "step2")
  .addEdge("step2", "step3");
```
:::

??? info "为什么要将应用程序步骤拆分为 LangGraph 序列？"
    LangGraph 可以轻松地为您的应用程序添加底层持久层。
    这允许在节点执行之间检查点状态，因此您的 LangGraph 节点负责：

- 如何 [检查点](../concepts/persistence.md) 状态更新
- 如何在 [人工参与](../concepts/human_in_the_loop.md) 工作流中恢复中断
- 如何使用 LangGraph 的 [时间旅行](../concepts/time-travel.md) 功能“回退”和分支执行

它们还决定了如何 [流式传输](../concepts/streaming.md) 执行步骤，以及如何使用 [LangGraph Studio](../concepts/langgraph_studio.md) 可视化和调试您的应用程序。

让我们演示一个端到端示例。我们将创建三个步骤的序列：

1. 填充 state 某个键的值
2. 更新相同的值
3. 填充另一个值

让我们首先定义我们的 [state](../concepts/low_level.md#state)。这会影响图的 [schema](../concepts/low_level.md#schema)，并且还可以指定如何应用更新。有关更多详细信息，请参阅 [本节](#process-state-updates-with-reducers)。

在本例中，我们将只跟踪两个值：

:::python
```python
from typing_extensions import TypedDict

class State(TypedDict):
    value_1: str
    value_2: int
```
:::

:::js
```typescript
import { z } from "zod";

const State = z.object({
  value1: z.string(),
  value2: z.number(),
});
```
:::

:::python
我们的 [nodes](../concepts/low_level.md#nodes) 只是 Python 函数，用于读取图的 state 并对其进行更新。此函数的第一个参数始终是 state：

```python
def step_1(state: State):
    return {"value_1": "a"}

def step_2(state: State):
    current_value_1 = state["value_1"]
    return {"value_1": f"{current_value_1} b"}

def step_3(state: State):
    return {"value_2": 10}
```
:::

:::js
我们的 [nodes](../concepts/low_level.md#nodes) 只是 TypeScript 函数，用于读取图的 state 并对其进行更新。此函数的第一个参数始终是 state：

```typescript
const step1 = (state: z.infer<typeof State>) => {
  return { value1: "a" };
};

const step2 = (state: z.infer<typeof State>) => {
  const currentValue1 = state.value1;
  return { value1: `${currentValue1} b` };
};

const step3 = (state: z.infer<typeof State>) => {
  return { value2: 10 };
};
```
:::

!!! note

    请注意，在发出 state 更新时，每个节点只需指定其希望更新的键的值。

    默认情况下，这将**覆盖**相应键的值。您还可以使用 [reducers](../concepts/low_level.md#reducers) 来控制更新的处理方式——例如，您可以改为将连续的更新附加到某个键。有关使用 reducers 更新 state 的更多详细信息，请参阅 [本节](#process-state-updates-with-reducers)。

最后，我们定义图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义一个操作此 state 的图。

:::python
我们将使用 [add_node](../concepts/low_level.md#messagesstate) 和 [add_edge](../concepts/low_level.md#edges) 来填充我们的图并定义其控制流。

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# 添加节点
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# 添加边
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```
:::

:::js
我们将使用 [addNode](../concepts/low_level.md#nodes) 和 [addEdge](../concepts/low_level.md#edges) 来填充我们的图并定义其控制流。

```typescript
import { START, StateGraph } from "@langchain/langgraph";

const graph = new StateGraph(State)
  .addNode("step1", step1)
  .addNode("step2", step2)
  .addNode("step3", step3)
  .addEdge(START, "step1")
  .addEdge("step1", "step2")
  .addEdge("step2", "step3")
  .compile();
```
:::

:::python
!!! tip "指定自定义名称"

    您可以使用 `.add_node` 为节点指定自定义名称：

    ```python
    builder.add_node("my_node", step_1)
    ```
:::

:::js
!!! tip "指定自定义名称"

    您可以使用 `.addNode` 为节点指定自定义名称：

    ```typescript
    const graph = new StateGraph(State)
      .addNode("myNode", step1)
      .compile();
    ```
:::

请注意：

:::python
- `.add_edge` 接受节点名称，默认情况下函数为 `node.__name__`。
- 我们必须指定图的入口点。为此，我们添加了一个带有 [START node](../concepts/low_level.md#start-node) 的边。
- 图在没有更多可执行节点时停止。

接下来我们 [编译](../concepts/low_level.md#compiling-your-graph) 我们的图。这会提供一些基本的图结构检查（例如，识别孤立节点）。如果我们通过 [checkpointer](../concepts/persistence.md) 为应用程序添加持久性，它也将在此处传入。

```python
graph = builder.compile()
```
:::

:::js
- `.addEdge` 接受节点名称，对于函数，默认为 `node.name`。
- 我们必须指定图的入口点。为此，我们添加了一个带有 [START node](../concepts/low_level.md#start-node) 的边。
- 图在没有更多可执行节点时停止。

接下来我们 [编译](../concepts/low_level.md#compiling-your-graph) 我们的图。这会提供一些基本的图结构检查（例如，识别孤立节点）。如果我们通过 [checkpointer](../concepts/persistence.md) 为应用程序添加持久性，它也将在此处传入。
:::

LangGraph 提供了用于可视化图的内置实用程序。让我们检查一下我们的序列。有关可视化的详细信息，请参见 [本指南](#visualize-your-graph)。

:::python
```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![步骤序列图](assets/graph_api_image_2.png)
:::

:::js
```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

让我们继续一个简单的调用：

:::python
```python
graph.invoke({"value_1": "c"})
```

```
{'value_1': 'a b', 'value_2': 10}
```
:::

:::js
```typescript
const result = await graph.invoke({ value1: "c" });
console.log(result);
```

```
{ value1: 'a b', value2: 10 }
```
:::

请注意：

- 我们通过提供单个 state 键的值来启动调用。我们必须至少提供一个键的值。
- 我们传入的值被第一个节点覆盖。
- 第二个节点更新了该值。
- 第三个节点填充了另一个值。

:::python
!!! tip "内置简写"

    `langgraph>=0.2.46` 包含一个内置的 `add_sequence` 简写，用于添加节点序列。您可以按照以下方式编译相同的图：

    ```python
    # highlight-next-line
    builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
    builder.add_edge(START, "step_1")

    graph = builder.compile()

    graph.invoke({"value_1": "c"})
    ```
:::

## 创建分支

节点的并行执行对于加速整体图操作至关重要。LangGraph 支持对节点的并行执行提供原生支持，这可以显著提高基于图的工作流的性能。这种并行化是通过扇入和扇出机制实现的，同时利用标准边和 [conditional_edges](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.MessageGraph.add_conditional_edges)。下面是一些示例，展示如何创建适合您的分支数据流。

### 并行运行图节点

在本例中，我们从 `Node A` 扇出到 `B 和 C`，然后扇入到 `D`。在我们的 state 中，[我们指定了 add 操作的 reducer](https://langchain-ai.github.io/langgraph/concepts/low_level.md#reducers)。这将合并或累积 State 中特定键的值，而不是简单地覆盖现有值。对于列表，这意味着将新列表与现有列表连接起来。有关使用 reducers 更新 state 的更多详细信息，请参阅上面关于 [state reducers](#process-state-updates-with-reducers) 的部分。

:::python
```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add reducer 函数使此列表仅可追加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  // reducer 使此列表仅可追加
  aggregate: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log(`Adding "A" to ${state.aggregate}`);
  return { aggregate: ["A"] };
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log(`Adding "B" to ${state.aggregate}`);
  return { aggregate: ["B"] };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log(`Adding "C" to ${state.aggregate}`);
  return { aggregate: ["C"] };
};

const nodeD = (state: z.infer<typeof State>) => {
  console.log(`Adding "D" to ${state.aggregate}`);
  return { aggregate: ["D"] };
};

const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addNode("c", nodeC)
  .addNode("d", nodeD)
  .addEdge(START, "a")
  .addEdge("a", "b")
  .addEdge("a", "c")
  .addEdge("b", "d")
  .addEdge("c", "d")
  .addEdge("d", END)
  .compile();
```
:::

:::python
```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![并行执行图](assets/graph_api_image_3.png)
:::

:::js
```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

使用 reducer，您可以发现每个节点添加的值都会被累加。

:::python
```python
graph.invoke({"aggregate": []}, {"configurable": {"thread_id": "foo"}})
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
```
:::

:::js
```typescript
const result = await graph.invoke({
  aggregate: [],
});
console.log(result);
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
{ aggregate: ['A', 'B', 'C', 'D'] }
```
:::

!!! note

    在上面的示例中，节点 `"b"` 和 `"c"` 并行执行在同一个 [superstep](../concepts/low_level.md#graphs) 中。因为它们在同一个步骤中，所以节点 `"d"` 在 `"b"` 和 `"c"` 都完成后执行。

    重要的是，并行 superstep 的更新可能不会以一致的顺序。如果您需要并行 superstep 更新的稳定、预定的顺序，您应该将输出写入 state 的单独字段，并附带用于排序的值。

??? note "异常处理？"

    LangGraph 在 [supersteps](../concepts/low_level.md#graphs) 内执行节点，这意味着虽然并行分支是并行执行的，但整个 superstep 是**事务性**的。如果这些分支中的任何一个引发异常，**所有**更新都不会应用于 state（整个 superstep 出现错误）。

    重要的是，在使用 [checkpointer](../concepts/persistence.md) 时，成功节点的 superstep 中的结果会被保存，并且在恢复时不会重复。

    如果您有易出错的操作（可能需要处理不稳定的 API 调用），LangGraph 提供了两种解决方案：

    1. 您可以在节点内编写常规 Python 代码来捕获和处理异常。
    2. 您可以设置一个 **[retry_policy](../reference/types.md#langgraph.types.RetryPolicy)** 来指示图重试引发某些类型异常的节点。仅重试失败的分支，因此您无需担心执行重复工作。

    结合使用这些方法，您可以执行并行执行并完全控制异常处理。

:::python

### 延迟节点执行

当您希望将节点的执行推迟到所有其他待处理任务完成后才执行时，延迟节点执行非常有用。这在分支长度不同时尤其值得关注，这在 map-reduce 流程等工作流中很常见。

上面示例展示了如何在每条路径只有一个步骤时进行扇出和扇入。但如果一个分支有一个以上的步骤呢？让我们在 `"b"` 分支中添加一个节点 `"b_2"`：

```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add reducer 函数使此列表仅可追加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def b_2(state: State):
    print(f'Adding "B_2" to {state["aggregate"]}')
    return {"aggregate": ["B_2"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(b_2)
builder.add_node(c)
# highlight-next-line
builder.add_node(d, defer=True)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "b_2")
builder.add_edge("b_2", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![延迟执行图](assets/graph_api_image_4.png)

```python
graph.invoke({"aggregate": []})
```

```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "B_2" to ['A', 'B', 'C']
Adding "D" to ['A', 'B', 'C', 'B_2']
```

在上面的示例中，节点 `"b"` 和 `"c"` 并行执行在同一个 superstep 中。我们将 `defer=True` 设置在节点 `d` 上，因此它在所有待处理任务完成后才会执行。在这种情况下，这意味着 `"d"` 会等待直到整个 `"b"` 分支完成执行。
:::

### 条件分支

:::python
如果您的扇出应该在运行时根据 state 而变化，您可以使用 [add_conditional_edges](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph.add_conditional_edges) 使用图 state 来选择一个或多个路径。请参见下面的示例，其中节点 `a` 生成一个 state 更新，该更新决定了下一个节点。

```python
import operator
from typing import Annotated, Literal, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    #向 state 添加一个键。我们将设置此键来确定
    #我们如何分支。
    which: str

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    # highlight-next-line
    return {"aggregate": ["A"], "which": "c"}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_edge(START, "a")
builder.add_edge("b", END)
builder.add_edge("c", END)

def conditional_edge(state: State) -> Literal["b", "c"]:
    #在此处填写使用 state
    #来确定下一个节点的任意逻辑
    return state["which"]

# highlight-next-line
builder.add_conditional_edges("a", conditional_edge)

graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![条件分支图](assets/graph_api_image_5.png)

```python
result = graph.invoke({"aggregate": []})
print(result)
```

```
Adding "A" to []
Adding "C" to ['A']
{'aggregate': ['A', 'C'], 'which': 'c'}
```
:::

:::js
如果您的扇出应该在运行时根据 state 而变化，您可以使用 [addConditionalEdges](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph.addConditionalEdges) 使用图 state 来选择一个或多个路径。请参见下面的示例，其中节点 `a` 生成一个 state 更新，该更新决定了下一个节点。

```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  aggregate: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
  // 向 state 添加一个键。我们将设置此键来确定
  // 我们如何分支。
  which: z.string().langgraph.reducer((x, y) => y ?? x),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log(`Adding "A" to ${state.aggregate}`);
  // highlight-next-line
  return { aggregate: ["A"], which: "c" };
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log(`Adding "B" to ${state.aggregate}`);
  return { aggregate: ["B"] };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log(`Adding "C" to ${state.aggregate}`);
  return { aggregate: ["C"] };
};

const conditionalEdge = (state: z.infer<typeof State>): "b" | "c" => {
  //在此处填写使用 state
  //来确定下一个节点的任意逻辑
  return state.which as "b" | "c";
};

// highlight-next-line
const graph = new StateGraph(State)
  .addNode("a", nodeA)  
  .addNode("b", nodeB)
  .addNode("c", nodeC)
  .addEdge(START, "a")
  .addEdge("b", END)
  .addEdge("c", END)
  .addConditionalEdges("a", conditionalEdge)
  .compile();
```

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```

```typescript
const result = await graph.invoke({ aggregate: [] });
console.log(result);
```

```
Adding "A" to []
Adding "C" to ['A']
{ aggregate: ['A', 'C'], which: 'c' }
```
:::

!!! tip

    您的条件边可以路由到多个目标节点。例如：

    :::python
    ```python
    def route_bc_or_cd(state: State) -> Sequence[str]:
        if state["which"] == "cd":
            return ["c", "d"]
        return ["b", "c"]
    ```
    :::

    :::js
    ```typescript
    const routeBcOrCd = (state: z.infer<typeof State>): string[] => {
      if (state.which === "cd") {
        return ["c", "d"];
      }
      return ["b", "c"];
    };
    ```
    :::

## Map-Reduce 和 Send API

LangGraph 支持使用 Send API 进行 map-reduce 和其他高级分支模式。以下是如何使用它的示例：

:::python
```python
from langgraph.graph import StateGraph, START, END
from langgraph.types import Send
from typing_extensions import TypedDict, Annotated
import operator

class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    jokes: Annotated[list[str], operator.add]
    best_selected_joke: str

def generate_topics(state: OverallState):
    return {"subjects": ["lions", "elephants", "penguins"]}

def generate_joke(state: OverallState):
    joke_map = {
        "lions": "Why don't lions like fast food? Because they can't catch it!",
        "elephants": "Why don't elephants use computers? They're afraid of the mouse!",
        "penguins": "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
    }
    return {"jokes": [joke_map[state["subject"]]]}

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

def best_joke(state: OverallState):
    return {"best_selected_joke": "penguins"}

builder = StateGraph(OverallState)
builder.add_node("generate_topics", generate_topics)
builder.add_node("generate_joke", generate_joke)
builder.add_node("best_joke", best_joke)
builder.add_edge(START, "generate_topics")
builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
builder.add_edge("generate_joke", "best_joke")
builder.add_edge("best_joke", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![带 fanout 的 Map-reduce 图](assets/graph_api_image_6.png)

```python
# 调用图：这里我们调用它来生成一个笑话列表
for step in graph.stream({"topic": "animals"}):
    print(step)
```

```
{'generate_topics': {'subjects': ['lions', 'elephants', 'penguins']}}
{'generate_joke': {'jokes': ["Why don't lions like fast food? Because they can't catch it!"]}}
{'generate_joke': {'jokes': ["Why don't elephants use computers? They're afraid of the mouse!"]}}
{'generate_joke': {'jokes': ['Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice.']}}
{'best_joke': {'best_selected_joke': 'penguins'}}
```
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END, Send } from "@langchain/langgraph";
import { z } from "zod";

const OverallState = z.object({
  topic: z.string(),
  subjects: z.array(z.string()),
  jokes: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
  bestSelectedJoke: z.string(),
});

const generateTopics = (state: z.infer<typeof OverallState>) => {
  return { subjects: ["lions", "elephants", "penguins"] };
};

const generateJoke = (state: { subject: string }) => {
  const jokeMap: Record<string, string> = {
    lions: "Why don't lions like fast food? Because they can't catch it!",
    elephants: "Why don't elephants use computers? They're afraid of the mouse!",
    penguins: "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
  };
  return { jokes: [jokeMap[state.subject]] };
};

const continueToJokes = (state: z.infer<typeof OverallState>) => {
  return state.subjects.map((subject) => new Send("generateJoke", { subject }));
};

const bestJoke = (state: z.infer<typeof OverallState>) => {
  return { bestSelectedJoke: "penguins" };
};

const graph = new StateGraph(OverallState)
  .addNode("generateTopics", generateTopics)
  .addNode("generateJoke", generateJoke)
  .addNode("bestJoke", bestJoke)
  .addEdge(START, "generateTopics")
  .addConditionalEdges("generateTopics", continueToJokes)
  .addEdge("generateJoke", "bestJoke")
  .addEdge("bestJoke", END)
  .compile();
```

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```

```typescript
// 调用图：这里我们调用它来生成一个笑话列表
for await (const step of await graph.stream({ topic: "animals" })) {
  console.log(step);
}
```

```
{ generateTopics: { subjects: [ 'lions', 'elephants', 'penguins' ] } }
{ generateJoke: { jokes: [ "Why don't lions like fast food? Because they can't catch it!" ] } }
{ generateJoke: { jokes: [ "Why don't elephants use computers? They're afraid of the mouse!" ] } }
{ generateJoke: { jokes: [ "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice." ] } }
{ bestJoke: { bestSelectedJoke: 'penguins' } }
```
:::

## 创建和控制循环

在创建带有关环的图时，我们需要一种终止执行的机制。这通常通过添加一个 [conditional edge](../concepts/low_level.md#conditional-edges) 来完成，该边在达到某个终止条件后会路由到 [END](../contacts/low_level.md#end-node) 节点。

您也可以在调用或流式传输图时设置图的递归限制。递归限制设置了图在引发错误之前允许执行的 [supersteps](../concepts/low_level.md#graphs) 数量。在此处阅读有关递归限制概念的更多信息 [here](../concepts/low_level.md#recursion-limit)。

让我们考虑一个带有简单循环的图，以更好地理解这些机制的工作原理。

!!! tip

    要返回 state 的最后一个值而不是收到递归限制错误，请参阅 [下一节](#impose-a-recursion-limit)。

创建循环时，您可以包含一个指定终止条件的条件边：

:::python
```python
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

def route(state: State) -> Literal["b", END]:
    if termination_condition(state):
        return END
    else:
        return "b"

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```
:::

:::js
```typescript
const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addEdge(START, "a")
  .addConditionalEdges("a", route)
  .addEdge("b", "a")
  .compile();

const route = (state: z.infer<typeof State>): "b" | typeof END => {
  if (terminationCondition(state)) {
    return END;
  } else {
    return "b";
  }
};
```
:::

要控制递归限制，请在配置中指定“recursionLimit”。这将引发一个 `GraphRecursionError`，您可以捕获并处理它：

:::python
```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke(inputs, {"recursion_limit": 3})
except GraphRecursionError:
    print("Recursion Error")
```
:::

:::js
```typescript
import { GraphRecursionError } from "@langchain/langgraph";

try {
  await graph.invoke(inputs, { recursionLimit: 3 });
} catch (error) {
  if (error instanceof GraphRecursionError) {
    console.log("Recursion Error");
  }
}
```
:::

让我们定义一个带有简单循环的图。请注意，我们使用条件边来实现终止条件。

:::python
```python
import operator
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add reducer 函数使此列表仅可追加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Node A sees {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Node B sees {state["aggregate"]}')
    return {"aggregate": ["B"]}

# 定义节点
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

# 定义边
def route(state: State) -> Literal["b", END]:
    if len(state["aggregate"]) < 7:
        return "b"
    else:
        return END

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![简单循环图](assets/graph_api_image_7.png)
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  // reducer 使此列表仅可追加
  aggregate: z.array(z.string()).langgraph.reducer((x, y) => x.concat(y)),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log(`Node A sees ${state.aggregate}`);
  return { aggregate: ["A"] };
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log(`Node B sees ${state.aggregate}`);
  return { aggregate: ["B"] };
};

// 定义边
const route = (state: z.infer<typeof State>): "b" | typeof END => {
  if (state.aggregate.length < 7) {
    return "b";
  } else {
    return END;
  }
};

const graph = new StateGraph(State)
  .addNode("a", nodeA)
  .addNode("b", nodeB)
  .addEdge(START, "a")
  .addConditionalEdges("a", route)
  .addEdge("b", "a")
  .compile();
```

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::

此架构类似于 [React agent](../agents/overview.md)，其中节点 "a" 是一个工具调用模型，节点 "b" 代表工具。

在我们的 `route` 条件边中，我们指定一旦 state 中的 `"aggregate"` 列表长度达到阈值，就结束。

调用图，我们看到在达到终止条件之前，节点 `"a"` 和 `"b"` 会交替执行。

:::python
```python
graph.invoke({"aggregate": []})
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Node B sees ['A', 'B', 'A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B', 'A', 'B']
```
:::

:::js
```typescript
const result = await graph.invoke({ aggregate: [] });
console.log(result);
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Node B sees ['A', 'B', 'A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B', 'A', 'B']
{ aggregate: ['A', 'B', 'A', 'B', 'A', 'B', 'A'] }
```
:::

### 施加递归限制

在某些应用程序中，我们可能无法保证函数会达到给定的终止条件。在这些情况下，我们可以设置图的 [recursion limit](../concepts/low_level.md#recursion-limit)。这将引发一个 `GraphRecursionError`，之后会有给定数量的 [supersteps](../concepts/low_level.md#graphs)。然后我们可以捕获并处理此异常：

:::python
```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke({"aggregate": []}, {"recursion_limit": 4})
except GraphRecursionError:
    print("Recursion Error")
```

```
Node A sees []
Node B sees ['A']
Node C sees ['A', 'B']
Node D sees ['A', 'B']
Node A sees ['A', 'B', 'C', 'D']
Recursion Error
```
:::

:::js
```typescript
import { GraphRecursionError } from "@langchain/langgraph";

try {
  await graph.invoke({ aggregate: [] }, { recursionLimit: 4 });
} catch (error) {
  if (error instanceof GraphRecursionError) {
    console.log("Recursion Error");
  }
}
```

```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Recursion Error
```
:::


:::python
??? example "扩展示例：在达到递归限制时返回 state"

    而不是引发 `GraphRecursionError`，我们可以向 state 中引入一个新键，该键跟踪达到递归限制前剩余的步数。然后我们可以使用此键来确定是否应结束运行。

    LangGraph 实现了一个特殊的 `RemainingSteps` 注释。在底层，它创建了一个 `ManagedValue` 通道——一个将在我们的图运行期间存在，之后不再存在的 state 通道。

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from langgraph.managed.is_last_step import RemainingSteps

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]
        remaining_steps: RemainingSteps

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    # 定义节点
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)

    # 定义边
    def route(state: State) -> Literal["b", END]:
        if state["remaining_steps"] <= 2:
            return END
        else:
            return "b"

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "a")
    graph = builder.compile()

    # 测试一下
    result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    print(result)
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node A sees ['A', 'B']
    {'aggregate': ['A', 'B', 'A']}
    ```
:::

:::python
??? example "扩展示例：分支循环"

    为了更好地理解递归限制的工作原理，让我们看一个更复杂的示例。下面我们实现一个循环，但其中一个步骤会扇出到两个节点：

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    def c(state: State):
        print(f'Node C sees {state["aggregate"]}')
        return {"aggregate": ["C"]}

    def d(state: State):
        print(f'Node D sees {state["aggregate"]}')
        return {"aggregate": ["D"]}

    # 定义节点
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)
    builder.add_node(c)
    builder.add_node(d)

    # 定义边
    def route(state: State) -> Literal["b", END]:
        if len(state["aggregate"]) < 7:
            return "b"
        else:
            return END

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "c")
    builder.add_edge("b", "d")
    builder.add_edge(["c", "d"], "a")
    graph = builder.compile()
    ```

    ```python
    from IPython.display import Image, display

    display(Image(graph.get_graph().draw_mermaid_png()))
    ```

    ![带分支的复杂循环图](assets/graph_api_image_8.png)

    这个图看起来很复杂，但可以看作是 [supersteps](../concepts/low_level.md#graphs) 的循环：

    1. Node A
    2. Node B
    3. Nodes C 和 D
    4. Node A
    5. ...

    我们有一个由四个 supersteps 组成的循环，其中节点 C 和 D 并行执行。

    像以前一样调用图，我们看到在达到终止条件之前完成了两个完整的“圈”：

    ```python
    result = graph.invoke({"aggregate": []})
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node D sees ['A', 'B']
    Node C sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Node B sees ['A', 'B', 'C', 'D', 'A']
    Node D sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node C sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node A sees ['A', 'B', 'C', 'D', 'A', 'B', 'C', 'D']
    ```

    但是，如果我们设置递归限制为 4，则只完成一圈，因为每一圈由四个 superstep 组成：

    ```python
    from langgraph.errors import GraphRecursionError

    try:
        result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    except GraphRecursionError:
        print("Recursion Error")
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node C sees ['A', 'B']
    Node D sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Recursion Error
    ```
:::

:::python

## Async

使用异步编程范例可以在并发执行 [IO-bound](https://en.wikipedia.org/wiki/I/O_bound) 代码时（例如，向聊天模型提供者进行并发 API 请求）带来显著的性能改进。

要将 `sync` 实现的图转换为 `async` 实现，您需要：

1. 将 `nodes` 更新为使用 `async def` 而不是 `def`。
2. 更新内部代码以正确使用 `await`。
3. 根据需要使用 `.ainvoke` 或 `.astream` 调用图。

由于许多 LangChain 对象实现了 [Runnable Protocol](https://python.langchain.com/docs/expression_language/interface/)，它具有所有 `sync` 方法的 `async` 变体，因此通常可以很快地将 `sync` 图升级到 `async` 图。

请参见下面的示例。为了演示 LLM 的异步调用，我们将包含一个聊天模型：

{% include-markdown "../../snippets/chat_model_tabs.md" %}

```python
from langchain.chat_models import init_chat_model
from langgraph.graph import MessagesState, StateGraph

# highlight-next-line
async def node(state: MessagesState): # (1)!
    # highlight-next-line
    new_message = await llm.ainvoke(state["messages"]) # (2)!
    return {"messages": [new_message]}

builder = StateGraph(MessagesState).add_node(node).set_entry_point("node")
graph = builder.compile()

input_message = {"role": "user", "content": "Hello"}
# highlight-next-line
result = await graph.ainvoke({"messages": [input_message]}) # (3)!
```

1. 将节点声明为异步函数。
2. 在节点内使用异步调用（如果可用）。
3. 直接在图对象本身上使用异步调用。

!!! tip "异步流式传输"

    有关异步流式传输的示例，请参见 [streaming guide](./streaming.md)。

:::

## 使用 `Command` 组合控制流和状态更新

将控制流（边）和状态更新（节点）组合起来可能很有用。例如，您可能希望在**同一个节点**中同时执行状态更新**并**决定下一步转到哪个节点。LangGraph 提供了一种方法，通过从函数返回 [Command](../reference/types.md#langgraph.types.Command) 对象来实现这一点：

:::python
```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # state update
        update={"foo": "bar"},
        # control flow
        goto="my_other_node"
    )
```
:::

:::js
```typescript
import { Command } from "@langchain/langgraph";

const myNode = (state: State): Command => {
  return new Command({
    update: { foo: "bar" },
    goto: "myOtherNode"
  });
};
```
:::

下面我们展示一个端到端示例。让我们创建一个包含 3 个节点的简单图：A、B 和 C。我们将首先执行节点 A，然后根据节点 A 的输出决定下一步转到节点 B 还是节点 C。

:::python
```python
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

# 定义图 state
class State(TypedDict):
    foo: str

# 定义节点

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    print("Called A")
    value = random.choice(["b", "c"])
    # 这是条件边函数的替代
    if value == "b":
        goto = "node_b"
    else:
        goto = "node_c"

    # 注意 Command 如何允许您同时更新图 state 并路由到下一个节点
    return Command(
        # 这是 state 更新
        update={"foo": value},
        # 这是边的替代
        goto=goto,
    )

def node_b(state: State):
    print("Called B")
    return {"foo": state["foo"] + "b"}

def node_c(state: State):
    print("Called C")
    return {"foo": state["foo"] + "c"}
```

我们现在可以用上述节点创建 `StateGraph`。请注意，该图没有用于路由的 [conditional edges](../concepts/low_level.md#conditional-edges)！这是因为控制流是在 `node_a` 中的 `Command` 中定义的。

```python
builder = StateGraph(State)
builder.add_edge(START, "node_a")
builder.add_node(node_a)
builder.add_node(node_b)
builder.add_node(node_c)
# 注意：节点 A、B 和 C 之间没有边！

graph = builder.compile()
```

!!! important

    您可能已经注意到，我们使用了 `Command` 作为返回类型注解，例如 `Command[Literal["node_b", "node_c"]]`。这对于图渲染是必需的，它告诉 LangGraph `node_a` 可以导航到 `node_b` 和 `node_c`。

```python
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

![基于 Command 的图导航](assets/graph_api_image_11.png)

如果我们多次运行该图，我们将看到它根据节点 A 中的随机选择采取不同的路径（A -> B 或 A -> C）。

```python
graph.invoke({"foo": ""})
```

```
Called A
Called C
```
:::

:::js
```typescript
import { StateGraph, START, Command } from "@langchain/langgraph";
import { z } from "zod";

// 定义图 state
const State = z.object({
  foo: z.string(),
});

// 定义节点

const nodeA = (state: z.infer<typeof State>): Command => {
  console.log("Called A");
  const value = Math.random() > 0.5 ? "nodeB" : "nodeC";
  // 这是条件边函数的替代  
  const goto = value === "b" ? "nodeB" : "nodeC";

  // 注意 Command 如何允许您同时更新图 state 并路由到下一个节点
  return new Command({
    update: { foo: value },
    goto,
  });
};

const nodeB = (state: z.infer<typeof State>) => {
  console.log("Called B");
  return { foo: state.foo + "b" };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log("Called C");
  return { foo: state.foo + "c" };
};
```

我们现在可以用上述节点创建 `StateGraph`。请注意，该图没有用于路由的 [conditional edges](../concepts/low_level.md#conditional-edges)！这是因为控制流是在 `nodeA` 中的 `Command` 中定义的。

```typescript
const graph = new StateGraph(State)
  .addNode("nodeA", nodeA, { ends: ["nodeB", "nodeC"] })
  .addNode("nodeB", nodeB)
  .addNode("nodeC", nodeC)
  .addEdge(START, "nodeA")
  .compile();
```

!!! important

    您可能已经注意到，我们使用了 `ends` 来指定 `nodeA` 可以导航到哪些节点。这对于图渲染是必需的，它告诉 LangGraph `nodeA` 可以导航到 `nodeB` 和 `nodeC`。

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```

如果我们多次运行该图，我们将看到它采取不同的路径（A -> B 或 A -> C），这取决于节点 A 中的随机选择。

```typescript
const result = await graph.invoke({ foo: "" });
console.log(result);
```

```
Called A
Called C
{ foo: 'cc' }
```
:::

### 导航到父图中的节点

如果您正在使用 [subgraphs](../concepts/subgraphs.md)，您可能希望从子图中的节点导航到另一个子图（即父图中的另一个节点）。为此，您可以在 `Command` 中指定 `graph=Command.PARENT`：

:::python
```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # 其中 `other_subgraph` 是父图中的一个节点
        graph=Command.PARENT
    )
```
:::

:::js
```typescript
const myNode = (state: State): Command => {
  return new Command({
    update: { foo: "bar" },
    goto: "otherSubgraph",  // 其中 `otherSubgraph` 是父图中的一个节点
    graph: Command.PARENT
  });
};
```
:::

让我们用上面的示例来演示这一点。我们将通过将上面示例中的 `nodeA` 更改为一个单节点图，然后将其添加为父图的子图来实现。

!!! important "使用 `Command.PARENT` 进行状态更新"

    当您从子图节点将更新发送到父图节点，并且该键同时存在于父图和子图的 [state schemas](../concepts/low_level.md#schema) 中时，您**必须**在父图 state 中为要更新的键定义一个 [reducer](../concepts/low_level.md#reducers)。请参阅下面的示例。

:::python
```python
import operator
from typing_extensions import Annotated

class State(TypedDict):
    # 注意：这里我们定义了一个 reducer
    # highlight-next-line
    foo: Annotated[str, operator.add]

def node_a(state: State):
    print("Called A")
    value = random.choice(["a", "b"])
    # 这是条件边函数的替代
    if value == "a":
        goto = "node_b"
    else:
        goto = "node_c"

    # 注意 Command 如何允许您同时更新图 state 并路由到下一个节点
    return Command(
        update={"foo": value},
        goto=goto,
        # 这会告诉 LangGraph 导航到父图中的 node_b 或 node_c
        # 注意：这将导航到相对于子图最近的父图
        # highlight-next-line
        graph=Command.PARENT,
    )

subgraph = StateGraph(State).add_node(node_a).add_edge(START, "node_a").compile()

def node_b(state: State):
    print("Called B")
    # 注意：由于我们定义了 reducer，我们不再需要手动附加
    # 新字符到现有的 'foo' 值。相反，reducer 将自动附加这些值
    # （通过 operator.add）
    # highlight-next-line
    return {"foo": "b"}

def node_c(state: State):
    print("Called C")
    # highlight-next-line
    return {"foo": "c"}

builder = StateGraph(State)
builder.add_edge(START, "subgraph")
builder.add_node("subgraph", subgraph)
builder.add_node(node_b)
builder.add_node(node_c)

graph = builder.compile()
```

```python
graph.invoke({"foo": ""})
```

```
Called A
Called C
```
:::

:::js
```typescript
import "@langchain/langgraph/zod";
import { StateGraph, START, Command } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  // 注意：这里我们定义了一个 reducer
  // highlight-next-line
  foo: z.string().langgraph.reducer((x, y) => x + y),
});

const nodeA = (state: z.infer<typeof State>) => {
  console.log("Called A");
  const value = Math.random() > 0.5 ? "nodeB" : "nodeC";
  
  // 注意 Command 如何允许您同时更新图 state 并路由到下一个节点
  return new Command({
    update: { foo: "a" },
    goto: value,
    // this tells LangGraph to navigate to nodeB or nodeC in the parent graph
    // NOTE: this will navigate to the closest parent graph relative to the subgraph
    // highlight-next-line
    graph: Command.PARENT,
  });
};

const subgraph = new StateGraph(State)
  .addNode("nodeA", nodeA, { ends: ["nodeB", "nodeC"] })
  .addEdge(START, "nodeA")
  .compile();

const nodeB = (state: z.infer<typeof State>) => {
  console.log("Called B");
  // 注意：由于我们定义了 reducer，我们不再需要手动附加
  // 新字符到现有的 'foo' 值。相反，reducer 将自动附加这些值
  // highlight-next-line
  return { foo: "b" };
};

const nodeC = (state: z.infer<typeof State>) => {
  console.log("Called C");
  // highlight-next-line
  return { foo: "c" };
};

const graph = new StateGraph(State)
  .addNode("subgraph", subgraph, { ends: ["nodeB", "nodeC"] })
  .addNode("nodeB", nodeB)
  .addNode("nodeC", nodeC)
  .addEdge(START, "subgraph")
  .compile();
```

```typescript
const result = await graph.invoke({ foo: "" });
console.log(result);
```

```
Called A
Called C
{ foo: 'ac' }
```
:::

### 在工具中使用

在工具中更新图状态是一个常见的用例。例如，在客户支持应用程序中，您可能希望在对话开始时根据客户的账号或 ID 查找客户信息。要从工具更新图 state，您可以从工具返回 `Command(update={"my_custom_key": "foo", "messages": [...]})`：

:::python
```python
@tool
def lookup_user_info(tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig):
    """Use this to look up user information to better assist them with their questions."""
    user_info = get_user_info(config.get("configurable", {}).get("user_id"))
    return Command(
        update={
            # update the state keys
            "user_info": user_info,
            # update the message history
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=tool_call_id)]
        }
    )
```
:::

:::js
```typescript
import { tool } from "@langchain/core/tools";
import { Command } from "@langchain/langgraph";
import { RunnableConfig } from "@langchain/core/runnables";
import { z } from "zod";

const lookupUserInfo = tool(
  async (input, config: RunnableConfig) => {
    const userId = config.configurable?.userId;
    const userInfo = getUserInfo(userId);
    return new Command({
      update: {
        // update the state keys
        userInfo: userInfo,
        // update the message history
        messages: [{
          role: "tool",
          content: "Successfully looked up user information",
          tool_call_id: config.toolCall.id
        }]
      }
    });
  },
  {
    name: "lookupUserInfo",
    description: "Use this to look up user information to better assist them with their questions.",
    schema: z.object({}),
  }
);
```
:::

!!! important

    当您从工具返回 `Command` 时，您**必须**在 `Command.update` 中同时包含 `messages`（或用于消息历史记录的任何 state 键），并且 `messages` 中的消息列表**必须**包含一个 `ToolMessage`。这是为了确保生成的消息历史记录有效（LLM 提供者要求 AI 消息带工具有号调用后跟工具结果消息）。

如果您使用的是通过 `Command` 更新状态的工具，我们建议使用预置的 [`ToolNode`](../reference/agents.md#langgraph.prebuilt.tool_node.ToolNode)，它会自动处理返回 `Command` 对象的工具，并将它们传播到图状态。如果您正在编写自定义节点来调用工具，您需要手动将工具返回的 `Command` 对象作为该节点的更新进行传播。

## 可视化您的图

这里我们演示如何可视化您创建的图。

您可以可视化任何任意的 [Graph](https://langchain-ai.github.io/langgraph/reference/graphs/)，包括 [StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.state.StateGraph)。

:::python
让我们玩得开心，画一些分形图 ：）。

```python
import random
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

class MyNode:
    def __init__(self, name: str):
        self.name = name
    def __call__(self, state: State):
        return {"messages": [("assistant", f"Called node {self.name}")]}

def route(state) -> Literal["entry_node", "__end__"]:
    if len(state["messages"]) > 10:
        return "__end__"
    return "entry_node"

def add_fractal_nodes(builder, current_node, level, max_level):
    if level > max_level:
        return
    # 在此级别创建的节点数
    num_nodes = random.randint(1, 3)  # 根据需要调整随机性
    for i in range(num_nodes):
        nm = ["A", "B", "C"][i]
        node_name = f"node_{current_node}_{nm}"
        builder.add_node(node_name, MyNode(node_name))
        builder.add_edge(current_node, node_name)
        # 递归添加更多节点
        r = random.random()
        if r > 0.2 and level + 1 < max_level:
            add_fractal_nodes(builder, node_name, level + 1, max_level)
        elif r > 0.05:
            builder.add_conditional_edges(node_name, route, node_name)
        else:
            # 结束
            builder.add_edge(node_name, "__end__")

def build_fractal_graph(max_level: int):
    builder = StateGraph(State)
    entry_point = "entry_node"
    builder.add_node(entry_point, MyNode(entry_point))
    builder.add_edge(START, entry_point)
    add_fractal_nodes(builder, entry_point, 1, max_level)
    # 可选：如果需要，设置完成点
    builder.add_edge(entry_point, END)  # 或任何特定节点
    return builder.compile()

app = build_fractal_graph(3)
```
:::

:::js
让我们创建一个简单的示例图来演示可视化。

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

const State = MessagesZodState.extend({
  value: z.number(),
});

const app = new StateGraph(State)
  .addNode("node1", (state) => {
    return { value: state.value + 1 };
  })
  .addNode("node2", (state) => {
    return { value: state.value * 2 };
  })
  .addEdge(START, "node1")
  .addConditionalEdges("node1", (state) => {
    if (state.value < 10) {
      return "node2";
    }
    return END;
  })
  .addEdge("node2", "node1")
  .compile();
```
:::

### Mermaid

我们也可以将图类转换为 Mermaid 语法。

:::python
```python
print(app.get_graph().draw_mermaid())
```

```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
	__start__([<p>__start__</p>]):::first
	entry_node(entry_node)
	node_entry_node_A(node_entry_node_A)
	node_entry_node_B(node_entry_node_B)
	node_node_entry_node_B_A(node_node_entry_node_B_A)
	node_node_entry_node_B_B(node_node_entry_node_B_B)
	node_node_entry_node_B_C(node_node_entry_node_B_C)
	__end__([<p>__end__</p>]):::last
	__start__ --> entry_node;
	entry_node --> __end__;
	entry_node --> node_entry_node_A;
	entry_node --> node_entry_node_B;
	node_entry_node_B --> node_node_entry_node_B_A;
	node_entry_node_B --> node_node_entry_node_B_B;
	node_entry_node_B --> node_node_entry_node_B_C;
	node_entry_node_A -.-> entry_node;
	node_entry_node_A -.-> __end__;
	node_node_entry_node_B_A -.-> entry_node;
	node_node_entry_node_B_A -.-> __end__;
	node_node_entry_node_B_B -.-> entry_node;
	node_node_entry_node_B_B -.-> __end__;
	node_node_entry_node_B_C -.-> entry_node;
	node_node_entry_node_B_C -.-> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
```
:::

:::js
```typescript
const drawableGraph = await app.getGraphAsync();
console.log(drawableGraph.drawMermaid());
```

```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
	__start__([<p>__start__</p>]):::first
	node1(node1)
	node2(node2)
	__end__([<p>__end__</p>]):::last
	__start__ --> node1;
	node1 -.-> node2;
	node1 -.-> __end__;
	node2 --> node1;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef first fill-opacity:0
	classDef last fill:#bfb6fc
```
:::

### PNG

:::python
如果需要，我们也可以将 Graph 渲染成 `.png`。这里我们可以选择三种方式：

- 使用 Mermaid.ink API（无需额外包）
- 使用 Mermaid + Pyppeteer（需要 `pip install pyppeteer`）
- 使用 graphviz（需要 `pip install graphviz`）

**使用 Mermaid.Ink**

默认情况下，`draw_mermaid_png()` 使用 Mermaid.Ink 的 API 来生成图表。

```python
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(app.get_graph().draw_mermaid_png()))
```

![分形图可视化](assets/graph_api_image_10.png)

**使用 Mermaid + Pyppeteer**

```python
import nest_asyncio

nest_asyncio.apply()  # 在 Jupyter Notebook 中运行异步函数需要

display(
    Image(
        app.get_graph().draw_mermaid_png(
            curve_style=CurveStyle.LINEAR,
            node_colors=NodeStyles(first="#ffdfba", last="#baffc9", default="#fad7de"),
            wrap_label_n_words=9,
            output_file_path=None,
            draw_method=MermaidDrawMethod.PYPPETEER,
            background_color="white",
            padding=10,
        )
    )
)
```

**使用 Graphviz**

```python
try:
    display(Image(app.get_graph().draw_png()))
except ImportError:
    print(
        "您可能需要安装 pygraphviz 的依赖项，更多信息请参见 https://github.com/pygraphviz/pygraphviz/blob/main/INSTALL.txt"
    )
```
:::

:::js
如果需要，我们也可以将 Graph 渲染成 `.png`。这使用了 Mermaid.ink API 来生成图表。

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await app.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("graph.png", imageBuffer);
```
:::