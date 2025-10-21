# 流式输出

您可以从 LangGraph agent 或工作流中[流式输出](../concepts/streaming.md)。

## 支持的流模式

:::python
将以下一个或多个流模式作为列表传递给 @[`stream()`][CompiledStateGraph.stream] 或 @[`astream()`][CompiledStateGraph.astream] 方法：
:::

:::js
将以下一个或多个流模式作为列表传递给 @[`stream()`][CompiledStateGraph.stream] 方法：
:::

| 模式       | 描述                                                                                                                                                                                    |
| ---------- | ----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `values`   | 在图的每一步之后流式传输状态的完整值。                                                                                                                                                 |
| `updates`  | 在图的每一步之后流式传输状态的更新。如果在同一步骤中进行了多次更新（例如，运行了多个节点），则这些更新会分开流式传输。                                                                                               |
| `custom`   | 从图节点内部流式传输自定义数据。                                                                                                                                                           |
| `messages` | 从调用 LLM 的任何图节点流式传输 2 元组（LLM token，元数据）。                                                                                                                               |
| `debug`    | 在图执行过程中流式传输尽可能多的信息。                                                                                                                                                       |

## 从 agent 流式传输

### Agent 进度

:::python
要流式传输 agent 进度，请使用 `stream_mode="updates"` 的 @[`stream()`][CompiledStateGraph.stream] 或 @[`astream()`][CompiledStateGraph.astream] 方法。这会在每次 agent 步骤后发出一个事件。
:::

:::js
要流式传输 agent 进度，请使用 `streamMode: "updates"` 的 @[`stream()`][CompiledStateGraph.stream] 方法。这会在每次 agent 步骤后发出一个事件。
:::

例如，如果您有一个调用一次工具的 agent，您应该会看到以下更新：

- **LLM 节点**：带有工具调用请求的 AI 消息
- **工具节点**：带有执行结果的工具消息
- **LLM 节点**：最终的 AI 响应

:::python
=== "同步"

    ```python
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )
    # highlight-next-line
    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode="updates"
    ):
        print(chunk)
        print("\n")
    ```

=== "异步"

    ```python
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )
    # highlight-next-line
    async for chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode="updates"
    ):
        print(chunk)
        print("\n")
    ```

:::

:::js

```typescript
const agent = createReactAgent({
  llm: model,
  tools: [getWeather],
});

for await (const chunk of await agent.stream(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  { streamMode: "updates" }
)) {
  console.log(chunk);
  console.log("\n");
}
```

:::

### LLM token

:::python
要流式传输 LLM 产生的 token，请使用 `stream_mode="messages"`：

=== "同步"

    ```python
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )
    # highlight-next-line
    for token, metadata in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode="messages"
    ):
        print("Token", token)
        print("Metadata", metadata)
        print("\n")
    ```

=== "异步"

    ```python
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )
    # highlight-next-line
    async for token, metadata in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode="messages"
    ):
        print("Token", token)
        print("Metadata", metadata)
        print("\n")
    ```

:::

:::js
要流式传输 LLM 产生的 token，请使用 `streamMode: "messages"`：

```typescript
const agent = createReactAgent({
  llm: model,
  tools: [getWeather],
});

for await (const [token, metadata] of await agent.stream(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  { streamMode: "messages" }
)) {
  console.log("Token", token);
  console.log("Metadata", metadata);
  console.log("\n");
}
```

:::

### 工具更新

:::python
要流式传输工具执行时的更新，您可以使用 @[get_stream_writer][get_stream_writer]。

=== "同步"

    ```python
    # highlight-next-line
    from langgraph.config import get_stream_writer

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        # highlight-next-line
        writer = get_stream_writer()
        # stream any arbitrary data
        # highlight-next-line
        writer(f"Looking up data for city: {city}")
        return f"It's always sunny in {city}!"

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )

    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode="custom"
    ):
        print(chunk)
        print("\n")
    ```

=== "异步"

    ```python
    # highlight-next-line
    from langgraph.config import get_stream_writer

    def get_weather(city: str) -> str:
        """Get weather for a given city."""
        # highlight-next-line
        writer = get_stream_writer()
        # stream any arbitrary data
        # highlight-next-line
        writer(f"Looking up data for city: {city}")
        return f"It's always sunny in {city}!"

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )

    async for chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode="custom"
    ):
        print(chunk)
        print("\n")
    ```

!!! 注意

      如果您在工具内添加 `get_stream_writer`，您将无法在 LangGraph 执行上下文之外调用该工具。

:::

:::js
要流式传输工具执行时的更新，您可以使用配置中的 `writer` 参数。

```typescript
import { LangGraphRunnableConfig } from "@langchain/langgraph";

const getWeather = tool(
  async (input, config: LangGraphRunnableConfig) => {
    // Stream any arbitrary data
    config.writer?.("Looking up data for city: " + input.city);
    return `It's always sunny in ${input.city}!`;
  },
  {
    name: "get_weather",
    description: "Get weather for a given city.",
    schema: z.object({
      city: z.string().describe("The city to get weather for."),
    }),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getWeather],
});

for await (const chunk of await agent.stream(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  { streamMode: "custom" }
)) {
  console.log(chunk);
  console.log("\n");
}
```

!!! 注意
      如果您将 `writer` 参数添加到您的工具中，您将无法在没有提供 writer 函数的情况下在 LangGraph 执行上下文之外调用该工具。
:::

### 流式传输多种模式

:::python
您可以将流模式指定为列表来一次流式传输多种模式：`stream_mode=["updates", "messages", "custom"]`：

=== "同步"

    ```python
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )

    for stream_mode, chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode=["updates", "messages", "custom"]
    ):
        print(chunk)
        print("\n")
    ```

=== "异步"

    ```python
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
    )

    async for stream_mode, chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        stream_mode=["updates", "messages", "custom"]
    ):
        print(chunk)
        print("\n")
    ```

:::

:::js
您可以将流模式指定为数组来一次流式传输多种模式：`streamMode: ["updates", "messages", "custom"]`：

```typescript
const agent = createReactAgent({
  llm: model,
  tools: [getWeather],
});

for await (const chunk of await agent.stream(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  { streamMode: ["updates", "messages", "custom"] }
)) {
  console.log(chunk);
  console.log("\n");
}
```

:::

### 禁用流式传输

在某些应用程序中，您可能需要禁用单个 token 的流式传输，以用于给定的模型。这在[多 agent](../agents/multi-agent.md) 系统中非常有用，可以控制哪些 agent 流式传输它们的输出。

请参阅[模型](../agents/models.md#disable-streaming)指南了解如何禁用流式传输。

## 从工作流流式传输

### 基本用法示例

:::python
LangGraph 图公开 @[`.stream()`][Pregel.stream] (同步) 和 @[`.astream()`][Pregel.astream] (异步) 方法，以生成流式输出作为迭代器。

=== "同步"

    ```python
    for chunk in graph.stream(inputs, stream_mode="updates"):
        print(chunk)
    ```

=== "异步"

    ```python
    async for chunk in graph.astream(inputs, stream_mode="updates"):
        print(chunk)
    ```

:::

:::js
LangGraph 图公开 @[`.stream()`][Pregel.stream] 方法，以生成流式输出作为迭代器。

```typescript
for await (const chunk of await graph.stream(inputs, {
  streamMode: "updates",
})) {
  console.log(chunk);
}
```

:::

??? example "扩展示例：流式传输更新"

      :::python
      ```python
      from typing import TypedDict
      from langgraph.graph import StateGraph, START, END

      class State(TypedDict):
          topic: str
          joke: str

      def refine_topic(state: State):
          return {"topic": state["topic"] + " and cats"}

      def generate_joke(state: State):
          return {"joke": f"This is a joke about {state['topic']}"}

      graph = (
          StateGraph(State)
          .add_node(refine_topic)
          .add_node(generate_joke)
          .add_edge(START, "refine_topic")
          .add_edge("refine_topic", "generate_joke")
          .add_edge("generate_joke", END)
          .compile()
      )

      # highlight-next-line
      for chunk in graph.stream( # (1)!
          {"topic": "ice cream"},
          # highlight-next-line
          stream_mode="updates", # (2)!
      ):
          print(chunk)
      ```

      1. `stream()` 方法返回一个生成流式输出的迭代器。
      2. 设置 `stream_mode="updates"` 以仅在每个节点后流式传输图状态的更新。其他流模式也可用。有关详细信息，请参阅[支持的流模式](#supported-stream-modes)。
      :::

      :::js
      ```typescript
      import { StateGraph, START, END } from "@langchain/langgraph";
      import { z } from "zod";

      const State = z.object({
        topic: z.string(),
        joke: z.string(),
      });

      const graph = new StateGraph(State)
        .addNode("refineTopic", (state) => {
          return { topic: state.topic + " and cats" };
        })
        .addNode("generateJoke", (state) => {
          return { joke: `This is a joke about ${state.topic}` };
        })
        .addEdge(START, "refineTopic")
        .addEdge("refineTopic", "generateJoke")
        .addEdge("generateJoke", END)
        .compile();

      for await (const chunk of await graph.stream(
        { topic: "ice cream" },
        { streamMode: "updates" } // (1)!
      )) {
        console.log(chunk);
      }
      ```

      1. 设置 `streamMode: "updates"` 以仅在每个节点后流式传输图状态的更新。其他流模式也可用。有关详细信息，请参阅[支持的流模式](#supported-stream-modes)。
      :::

      ```output
      {'refineTopic': {'topic': 'ice cream and cats'}}
      {'generateJoke': {'joke': 'This is a joke about ice cream and cats'}}
      ```                                                                                                   |

### 流式传输多种模式

:::python
您可以将列表作为 `stream_mode` 参数传递，以一次流式传输多种模式。

流式传输的输出将是 `(mode, chunk)` 的元组，其中 `mode` 是流模式的名称，`chunk` 是该模式流式传输的数据。

=== "同步"

    ```python
    for mode, chunk in graph.stream(inputs, stream_mode=["updates", "custom"]):
        print(chunk)
    ```

=== "异步"

    ```python
    async for mode, chunk in graph.astream(inputs, stream_mode=["updates", "custom"]):
        print(chunk)
    ```

:::

:::js
您可以将数组作为 `streamMode` 参数传递，以一次流式传输多种模式。

流式传输的输出将是 `[mode, chunk]` 的元组，其中 `mode` 是流模式的名称，`chunk` 是该模式流式传输的数据。

```typescript
for await (const [mode, chunk] of await graph.stream(inputs, {
  streamMode: ["updates", "custom"],
})) {
  console.log(chunk);
}
```

:::

### 流式传输图状态

使用 `updates` 和 `values` 流模式，在图执行时流式传输图的状态。

- `updates` 在每个步骤后流式传输状态的**更新**。
- `values` 在每个步骤后流式传输状态的**完整值**。

:::python

```python
from typing import TypedDict
from langgraph.graph import StateGraph, START, END


class State(TypedDict):
  topic: str
  joke: str


def refine_topic(state: State):
    return {"topic": state["topic"] + " and cats"}


def generate_joke(state: State):
    return {"joke": f"This is a joke about {state['topic']}"}

graph = (
  StateGraph(State)
  .add_node(refine_topic)
  .add_node(generate_joke)
  .add_edge(START, "refine_topic")
  .add_edge("refine_topic", "generate_joke")
  .add_edge("generate_joke", END)
  .compile()
)
```

:::

:::js

```typescript
import { StateGraph, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  topic: z.string(),
  joke: z.string(),
});

const graph = new StateGraph(State)
  .addNode("refineTopic", (state) => {
    return { topic: state.topic + " and cats" };
  })
  .addNode("generateJoke", (state) => {
    return { joke: `This is a joke about ${state.topic}` };
  })
  .addEdge(START, "refineTopic")
  .addEdge("refineTopic", "generateJoke")
  .addEdge("generateJoke", END)
  .compile();
```

:::

=== "updates"

    使用此选项仅流式传输每个节点执行后返回的**状态更新**。流式传输的输出包括节点名称以及更新内容。

    :::python
    ```python
    for chunk in graph.stream(
        {"topic": "ice cream"},
        # highlight-next-line
        stream_mode="updates",
    ):
        print(chunk)
    ```
    :::

    :::js
    ```typescript
    for await (const chunk of await graph.stream(
      { topic: "ice cream" },
      { streamMode: "updates" }
    )) {
      console.log(chunk);
    }
    ```
    :::

=== "values"

    使用此选项流式传输每个节点执行后的图的**完整状态**。

    :::python
    ```python
    for chunk in graph.stream(
        {"topic": "ice cream"},
        # highlight-next-line
        stream_mode="values",
    ):
        print(chunk)
    ```
    :::

    :::js
    ```typescript
    for await (const chunk of await graph.stream(
      { topic: "ice cream" },
      { streamMode: "values" }
    )) {
      console.log(chunk);
    }
    ```
    :::

### 流式传输子图输出

:::python
要将[子图](../concepts/subgraphs.md)的输出包含在流式输出中，您可以在父图的 `.stream()` 方法中设置 `subgraphs=True`。这将流式传输来自父图和任何子图的输出。

输出将作为 `(namespace, data)` 的元组流式传输，其中 `namespace` 是一个元组，包含调用子图的节点的路径，例如 `("parent_node:<task_id>", "child_node:<task_id>")`。

```python
for chunk in graph.stream(
    {"foo": "foo"},
    # highlight-next-line
    subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. 设置 `subgraphs=True` 来流式传输子图的输出。
   :::

:::js
要将[子图](../concepts/subgraphs.md)的输出包含在流式输出中，您可以在父图的 `.stream()` 方法中设置 `subgraphs: true`。这将流式传输来自父图和任何子图的输出。

输出将作为 `[namespace, data]` 的元组流式传输，其中 `namespace` 是一个元组，包含调用子图的节点的路径，例如 `["parent_node:<task_id>", "child_node:<task_id>"]`。

```typescript
for await (const chunk of await graph.stream(
  { foo: "foo" },
  {
    subgraphs: true, // (1)!
    streamMode: "updates",
  }
)) {
  console.log(chunk);
}
```

1. 设置 `subgraphs: true` 来流式传输子图的输出。
   :::

??? example "扩展示例：从子图流式传输"

      :::python
      ```python
      from langgraph.graph import START, StateGraph
      from typing import TypedDict

      # Define subgraph
      class SubgraphState(TypedDict):
          foo: str  # note that this key is shared with the parent graph state
          bar: str

      def subgraph_node_1(state: SubgraphState):
          return {"bar": "bar"}

      def subgraph_node_2(state: SubgraphState):
          return {"foo": state["foo"] + state["bar"]}

      subgraph_builder = StateGraph(SubgraphState)
      subgraph_builder.add_node(subgraph_node_1)
      subgraph_builder.add_node(subgraph_node_2)
      subgraph_builder.add_edge(START, "subgraph_node_1")
      subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
      subgraph = subgraph_builder.compile()

      # Define parent graph
      class ParentState(TypedDict):
          foo: str

      def node_1(state: ParentState):
          return {"foo": "hi! " + state["foo"]}

      builder = StateGraph(ParentState)
      builder.add_node("node_1", node_1)
      builder.add_node("node_2", subgraph)
      builder.add_edge(START, "node_1")
      builder.add_edge("node_1", "node_2")
      graph = builder.compile()

      for chunk in graph.stream(
          {"foo": "foo"},
          stream_mode="updates",
          # highlight-next-line
          subgraphs=True, # (1)!
      ):
          print(chunk)
      ```

      1. 设置 `subgraphs=True` 来流式传输子图的输出。
      :::

      :::js
      ```typescript
      import { StateGraph, START } from "@langchain/langgraph";
      import { z } from "zod";

      // Define subgraph
      const SubgraphState = z.object({
        foo: z.string(), // note that this key is shared with the parent graph state
        bar: z.string(),
      });

      const subgraphBuilder = new StateGraph(SubgraphState)
        .addNode("subgraphNode1", (state) => {
          return { bar: "bar" };
        })
        .addNode("subgraphNode2", (state) => {
          return { foo: state.foo + state.bar };
        })
        .addEdge(START, "subgraphNode1")
        .addEdge("subgraphNode1", "subgraphNode2");
      const subgraph = subgraphBuilder.compile();

      // Define parent graph
      const ParentState = z.object({
        foo: z.string(),
      });

      const builder = new StateGraph(ParentState)
        .addNode("node1", (state) => {
          return { foo: "hi! " + state.foo };
        })
        .addNode("node2", subgraph)
        .addEdge(START, "node1")
        .addEdge("node1", "node2");
      const graph = builder.compile();

      for await (const chunk of await graph.stream(
        { foo: "foo" },
        {
          streamMode: "updates",
          subgraphs: true, // (1)!
        }
      )) {
        console.log(chunk);
      }
      ```

      1. 设置 `subgraphs: true` 来流式传输子图的输出。
      :::

      :::python
      ```
      ((), {'node_1': {'foo': 'hi! foo'}})
      (('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',), {'subgraph_node_1': {'bar': 'bar'}})
      (('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
      ((), {'node_2': {'foo': 'hi! foobar'}})
      ```
      :::

      :::js
      ```
      [[], {'node1': {'foo': 'hi! foo'}}]
      [['node2:dfddc4ba-c3c5-6887-5012-a243b5b377c2'], {'subgraphNode1': {'bar': 'bar'}}]
      [['node2:dfddc4ba-c3c5-6887-5012-a243b5b377c2'], {'subgraphNode2': {'foo': 'hi! foobar'}}]
      [[], {'node2': {'foo': 'hi! foobar'}}]
      ```
      :::

      **请注意**，我们不仅收到了节点更新，还收到了命名空间，它们告诉我们正在从哪个图（或子图）进行流式传输。

### 调试 {#debug}

使用 `debug` 流模式在图执行过程中流式传输尽可能多的信息。流式传输的输出包括节点名称以及完整状态。

:::python

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    # highlight-next-line
    stream_mode="debug",
):
    print(chunk)
```

:::

:::js

```typescript
for await (const chunk of await graph.stream(
  { topic: "ice cream" },
  { streamMode: "debug" }
)) {
  console.log(chunk);
}
```

:::

### LLM token {#messages}

使用 `messages` 流模式，可以**逐 token** 流式传输 Large Language Model (LLM) 的输出，这些输出可以来自图中的任何部分，包括节点、工具、子图或任务。

:::python
[`messages` 模式](#supported-stream-modes) 的流式输出是一个 `(message_chunk, metadata)` 元组，其中：

- `message_chunk`：来自 LLM 的 token 或消息片段。
- `metadata`：一个包含图节点和 LLM 调用详细信息的字典。

> 如果您的 LLM 无法集成到 LangChain 中，您可以使用 `custom` 模式来流式传输其输出。有关详细信息，请参阅[与任何 LLM 配合使用](#use-with-any-llm)。

!!! 警告 "Python < 3.11 的异步需要手动配置"

    当使用 Python < 3.11 并结合异步代码时，您必须显式地将 `RunnableConfig` 传递给 `ainvoke()` 以启用正确的流式传输。有关详细信息，请参阅[Python < 3.11 的异步](#async)或升级到 Python 3.11+。

```python
from dataclasses import dataclass

from langchain.chat_models import init_chat_model
from langgraph.graph import StateGraph, START


@dataclass
class MyState:
    topic: str
    joke: str = ""


llm = init_chat_model(model="openai:gpt-4o-mini")

def call_model(state: MyState):
    """Call the LLM to generate a joke about a topic"""
    # highlight-next-line
    llm_response = llm.invoke( # (1)!
        [
            {"role": "user", "content": f"Generate a joke about {state.topic}"}
        ]
    )
    return {"joke": llm_response.content}

graph = (
    StateGraph(MyState)
    .add_node(call_model)
    .add_edge(START, "call_model")
    .compile()
)

for message_chunk, metadata in graph.stream( # (2)!
    {"topic": "ice cream"},
    # highlight-next-line
    stream_mode="messages",
):
    if message_chunk.content:
        print(message_chunk.content, end="|", flush=True)
```

1. 请注意，即使 LLM 是使用 `.invoke` 而不是 `.stream` 运行的，也会发出消息事件。
2. "messages" 流模式返回一个 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式传输的 token，`metadata` 是一个包含 LLM 被调用的图节点信息和其他信息的字典。
   :::

:::js
[`messages` 模式](#supported-stream-modes) 的流式输出是一个 `[message_chunk, metadata]` 元组，其中：

- `message_chunk`：来自 LLM 的 token 或消息片段。
- `metadata`：一个包含图节点和 LLM 调用详细信息的字典。

> 如果您的 LLM 无法集成到 LangChain 中，您可以使用 `custom` 模式来流式传输其输出。有关详细信息，请参阅[与任何 LLM 配合使用](#use-with-any-llm)。

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { StateGraph, START } from "@langchain/langgraph";
import { z } from "zod";

const MyState = z.object({
  topic: z.string(),
  joke: z.string().default(""),
});

const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

const callModel = async (state: z.infer<typeof MyState>) => {
  // Call the LLM to generate a joke about a topic
  const llmResponse = await llm.invoke([
    { role: "user", content: `Generate a joke about ${state.topic}` },
  ]); // (1)!
  return { joke: llmResponse.content };
};

const graph = new StateGraph(MyState)
  .addNode("callModel", callModel)
  .addEdge(START, "callModel")
  .compile();

for await (const [messageChunk, metadata] of await graph.stream(
  // (2)!
  { topic: "ice cream" },
  { streamMode: "messages" }
)) {
  if (messageChunk.content) {
    console.log(messageChunk.content + "|");
  }
}
```

1. 请注意，即使 LLM 是使用 `.invoke` 而不是 `.stream` 运行的，也会发出消息事件。
2. "messages" 流模式返回一个 `[messageChunk, metadata]` 元组的迭代器，其中 `messageChunk` 是 LLM 流式传输的 token，`metadata` 是一个包含 LLM 被调用的图节点信息和其他信息的字典。
   :::

#### 按 LLM 调用进行过滤

您可以为 LLM 调用关联 `tags`，以按 LLM 调用过滤流式传输的 token。

:::python

```python
from langchain.chat_models import init_chat_model

llm_1 = init_chat_model(model="openai:gpt-4o-mini", tags=['joke']) # (1)!
llm_2 = init_chat_model(model="openai:gpt-4o-mini", tags=['poem']) # (2)!

graph = ... # define a graph that uses these LLMs

async for msg, metadata in graph.astream(  # (3)!
    {"topic": "cats"},
    # highlight-next-line
    stream_mode="messages",
):
    if metadata["tags"] == ["joke"]: # (4)!
        print(msg.content, end="|", flush=True)
```

1. llm_1 被标记为“joke”。
2. llm_2 被标记为“poem”。
3. `stream_mode` 设置为“messages”以流式传输 LLM token。`metadata` 包含 LLM 调用信息，包括标签。
4. 通过元数据中的 `tags` 字段过滤流式传输的 token，仅包括带有“joke”标签的 LLM 调用的 token。
   :::

:::js

```typescript
import { ChatOpenAI } from "@langchain/openai";

const llm1 = new ChatOpenAI({
  model: "gpt-4o-mini",
  tags: ['joke'] // (1)!
});
const llm2 = new ChatOpenAI({
  model: "gpt-4o-mini",
  tags: ['poem'] // (2)!
});

const graph = // ... define a graph that uses these LLMs

for await (const [msg, metadata] of await graph.stream( // (3)!
  { topic: "cats" },
  { streamMode: "messages" }
)) {
  if (metadata.tags?.includes("joke")) { // (4)!
    console.log(msg.content + "|");
  }
}
```

1. llm1 被标记为“joke”。
2. llm2 被标记为“poem”。
3. `streamMode` 设置为“messages”以流式传输 LLM token。`metadata` 包含 LLM 调用信息，包括标签。
4. 通过元数据中的 `tags` 字段过滤流式传输的 token，仅包括带有“joke”标签的 LLM 调用的 token。
   :::

??? example "扩展示例：按标签过滤"

      :::python
      ```python
      from typing import TypedDict

      from langchain.chat_models import init_chat_model
      from langgraph.graph import START, StateGraph

      joke_model = init_chat_model(model="openai:gpt-4o-mini", tags=["joke"]) # (1)!
      poem_model = init_chat_model(model="openai:gpt-4o-mini", tags=["poem"]) # (2)!


      class State(TypedDict):
            topic: str
            joke: str
            poem: str


      async def call_model(state, config):
            topic = state["topic"]
            print("Writing joke...")
            # Note: Passing the config through explicitly is required for python < 3.11
            # Since context var support wasn't added before then: https://docs.python.org/3/library/asyncio-task.html#creating-tasks
            joke_response = await joke_model.ainvoke(
                  [{"role": "user", "content": f"Write a joke about {topic}"}],
                  config, # (3)!
            )
            print("\n\nWriting poem...")
            poem_response = await poem_model.ainvoke(
                  [{"role": "user", "content": f"Write a short poem about {topic}"}],
                  config, # (3)!
            )
            return {"joke": joke_response.content, "poem": poem_response.content}


      graph = (
            StateGraph(State)
            .add_node(call_model)
            .add_edge(START, "call_model")
            .compile()
      )

      async for msg, metadata in graph.astream(
            {"topic": "cats"},
            # highlight-next-line
            stream_mode="messages", # (4)!
      ):
          if metadata["tags"] == ["joke"]: # (4)!
              print(msg.content, end="|", flush=True)
      ```

      1. `joke_model` 被标记为“joke”。
      2. `poem_model` 被标记为“poem”。
      3. `config` 被显式传递以确保正确传播上下文变量。在使用异步代码的 Python < 3.11 中，这是必需的。有关更多详细信息，请参阅[异步部分](#async)。
      4. `stream_mode` 设置为“messages”以流式传输 LLM token。`metadata` 包含 LLM 调用信息，包括标签。
      :::

      :::js
      ```typescript
      import { ChatOpenAI } from "@langchain/openai";
      import { StateGraph, START } from "@langchain/langgraph";
      import { z } from "zod";

      const jokeModel = new ChatOpenAI({
        model: "gpt-4o-mini",
        tags: ["joke"] // (1)!
      });
      const poemModel = new ChatOpenAI({
        model: "gpt-4o-mini",
        tags: ["poem"] // (2)!
      });

      const State = z.object({
        topic: z.string(),
        joke: z.string(),
        poem: z.string(),
      });

      const graph = new StateGraph(State)
        .addNode("callModel", (state) => {
          const topic = state.topic;
          console.log("Writing joke...");

          const jokeResponse = await jokeModel.invoke([
            { role: "user", content: `Write a joke about ${topic}` }
          ]);

          console.log("\n\nWriting poem...");
          const poemResponse = await poemModel.invoke([
            { role: "user", content: `Write a short poem about ${topic}` }
          ]);

          return {
            joke: jokeResponse.content,
            poem: poemResponse.content
          };
        })
        .addEdge(START, "callModel")
        .compile();

      for await (const [msg, metadata] of await graph.stream(
        { topic: "cats" },
        { streamMode: "messages" } // (3)!
      )) {
        if (metadata.tags?.includes("joke")) { // (4)!
          console.log(msg.content + "|");
        }
      }
      ```

      1. `jokeModel` 被标记为“joke”。
      2. `poemModel` 被标记为“poem”。
      3. `streamMode` 设置为“messages”以流式传输 LLM token。`metadata` 包含 LLM 调用信息，包括标签。
      4. 通过元数据中的 `tags` 字段过滤流式传输的 token，仅包括带有“joke”标签的 LLM 调用的 token。
      :::

#### 按节点过滤

要仅从特定节点流式传输 token，请使用 `stream_mode="messages"` 并通过流式传输元数据中的 `langgraph_node` 字段进行过滤：

:::python

```python
for msg, metadata in graph.stream( # (1)!
    inputs,
    # highlight-next-line
    stream_mode="messages",
):
    # highlight-next-line
    if msg.content and metadata["langgraph_node"] == "some_node_name": # (2)!
        ...
```

1. "messages" 流模式返回一个 `(message_chunk, metadata)` 元组，其中 `message_chunk` 是 LLM 流式传输的 token，`metadata` 是一个包含 LLM 被调用的图节点信息和其他信息的字典。
2. 通过元数据中的 `langgraph_node` 字段过滤流式传输的 token，仅包括来自 `write_poem` 节点的 token。
   :::

:::js

```typescript
for await (const [msg, metadata] of await graph.stream(
  // (1)!
  inputs,
  { streamMode: "messages" }
)) {
  if (msg.content && metadata.langgraph_node === "some_node_name") {
    // (2)!
    // ...
  }
}
```

1. "messages" 流模式返回一个 `[messageChunk, metadata]` 元组，其中 `messageChunk` 是 LLM 流式传输的 token，`metadata` 是一个包含 LLM 被调用的图节点信息和其他信息的字典。
2. 通过元数据中的 `langgraph_node` 字段过滤流式传输的 token，仅包括来自 `writePoem` 节点的 token。
   :::

??? example "扩展示例：从特定节点流式传输 LLM token"

      :::python
      ```python
      from typing import TypedDict
      from langgraph.graph import START, StateGraph
      from langchain_openai import ChatOpenAI

      model = ChatOpenAI(model="gpt-4o-mini")


      class State(TypedDict):
            topic: str
            joke: str
            poem: str


      def write_joke(state: State):
            topic = state["topic"]
            joke_response = model.invoke(
                  [{"role": "user", "content": f"Write a joke about {topic}"}]
            )
            return {"joke": joke_response.content}


      def write_poem(state: State):
            topic = state["topic"]
            poem_response = model.invoke(
                  [{"role": "user", "content": f"Write a short poem about {topic}"}]
            )
            return {"poem": poem_response.content}


      graph = (
            StateGraph(State)
            .add_node(write_joke)
            .add_node(write_poem)
            # write both the joke and the poem concurrently
            .add_edge(START, "write_joke")
            .add_edge(START, "write_poem")
            .compile()
      )

      # highlight-next-line
      for msg, metadata in graph.stream( # (1)!
          {"topic": "cats"},
          stream_mode="messages",
      ):
          # highlight-next-line
          if msg.content and metadata["langgraph_node"] == "write_poem": # (2)!
              print(msg.content, end="|", flush=True)
      ```

      1. "messages" 流模式返回一个 `(message_chunk, metadata)` 元组，其中 `message_chunk` 是 LLM 流式传输的 token，`metadata` 是一个包含 LLM 被调用的图节点信息和其他信息的字典。
      2. 通过元数据中的 `langgraph_node` 字段过滤流式传输的 token，仅包括来自 `write_poem` 节点的 token。
      :::

      :::js
      ```typescript
      import { ChatOpenAI } from "@langchain/openai";
      import { StateGraph, START } from "@langchain/langgraph";
      import { z } from "zod";

      const model = new ChatOpenAI({ model: "gpt-4o-mini" });

      const State = z.object({
        topic: z.string(),
        joke: z.string(),
        poem: z.string(),
      });

      const graph = new StateGraph(State)
        .addNode("writeJoke", async (state) => {
          const topic = state.topic;
          const jokeResponse = await model.invoke([
            { role: "user", content: `Write a joke about ${topic}` }
          ]);
          return { joke: jokeResponse.content };
        })
        .addNode("writePoem", async (state) => {
          const topic = state.topic;
          const poemResponse = await model.invoke([
            { role: "user", content: `Write a short poem about ${topic}` }
          ]);
          return { poem: poemResponse.content };
        })
        // write both the joke and the poem concurrently
        .addEdge(START, "writeJoke")
        .addEdge(START, "writePoem")
        .compile();

      for await (const [msg, metadata] of await graph.stream( // (1)!
        { topic: "cats" },
        { streamMode: "messages" }
      )) {
        if (msg.content && metadata.langgraph_node === "writePoem") { // (2)!
          console.log(msg.content + "|");
        }
      }
      ```

      1. "messages" 流模式返回一个 `[messageChunk, metadata]` 元组，其中 `messageChunk` 是 LLM 流式传输的 token，`metadata` 是一个包含 LLM 被调用的图节点信息和其他信息的字典。
      2. 通过元数据中的 `langgraph_node` 字段过滤流式传输的 token，仅包括来自 `writePoem` 节点的 token。
      :::

### 流式传输自定义数据

:::python
要从 LangGraph 节点或工具内部发送**自定义用户定义数据**，请按照以下步骤操作：

1. 使用 `get_stream_writer()` 访问流写入器并发出自定义数据。
2. 调用 `.stream()` 或 `.astream()` 时设置 `stream_mode="custom"`，以在流中获取自定义数据。您可以组合多种模式（例如 `["updates", "custom"]`），但至少一种必须是 `"custom"`。

!!! 警告 "Python < 3.11 的异步中没有 `get_stream_writer()`"

    在 Python < 3.11 上运行的异步代码中，`get_stream_writer()` 将无法工作。
    而是向您的节点或工具添加 `writer` 参数，并手动传递它。
    有关用法示例，请参阅[Python < 3.11 的异步](#async)。

=== "node"

      ```python
      from typing import TypedDict
      from langgraph.config import get_stream_writer
      from langgraph.graph import StateGraph, START

      class State(TypedDict):
          query: str
          answer: str

      def node(state: State):
          writer = get_stream_writer()  # (1)!
          writer({"custom_key": "Generating custom data inside node"}) # (2)!
          return {"answer": "some data"}

      graph = (
          StateGraph(State)
          .add_node(node)
          .add_edge(START, "node")
          .compile()
      )

      inputs = {"query": "example"}

      # Usage
      for chunk in graph.stream(inputs, stream_mode="custom"):  # (3)!
          print(chunk)
      ```

      1. 获取流写入器以发送自定义数据。
      2. 发出自定义键值对（例如，进度更新）。
      3. 设置 `stream_mode="custom"` 以在流中接收自定义数据。

=== "tool"

      ```python
      from langchain_core.tools import tool
      from langgraph.config import get_stream_writer

      @tool
      def query_database(query: str) -> str:
          """Query the database."""
          writer = get_stream_writer() # (1)!
          # highlight-next-line
          writer({"data": "Retrieved 0/100 records", "type": "progress"}) # (2)!
          # perform query
          # highlight-next-line
          writer({"data": "Retrieved 100/100 records", "type": "progress"}) # (3)!
          return "some-answer"


      graph = ... # define a graph that uses this tool

      for chunk in graph.stream(inputs, stream_mode="custom"): # (4)!
          print(chunk)
      ```

      1. 访问流写入器以发送自定义数据。
      2. 发出自定义键值对（例如，进度更新）。
      3. 发出另一个自定义键值对。
      4. 设置 `stream_mode="custom"` 以在流中接收自定义数据。

:::

:::js
要从 LangGraph 节点或工具内部发送**自定义用户定义数据**，请按照以下步骤操作：

1. 使用 `LangGraphRunnableConfig` 中的 `writer` 参数发出自定义数据。
2. 调用 `.stream()` 时设置 `streamMode: "custom"`，以在流中获取自定义数据。您可以组合多种模式（例如 `["updates", "custom"]`），但至少一种必须是 `"custom"`。

=== "node"

      ```typescript
      import { StateGraph, START, LangGraphRunnableConfig } from "@langchain/langgraph";
      import { z } from "zod";

      const State = z.object({
        query: z.string(),
        answer: z.string(),
      });

      const graph = new StateGraph(State)
        .addNode("node", async (state, config) => {
          config.writer({ custom_key: "Generating custom data inside node" }); // (1)!
          return { answer: "some data" };
        })
        .addEdge(START, "node")
        .compile();

      const inputs = { query: "example" };

      // Usage
      for await (const chunk of await graph.stream(inputs, { streamMode: "custom" })) { // (2)!
        console.log(chunk);
      }
      ```

      1. 使用 writer 发出自定义键值对（例如，进度更新）。
      2. 设置 `streamMode: "custom"` 以在流中接收自定义数据。

=== "tool"

      ```typescript
      import { tool } from "@langchain/core/tools";
      import { LangGraphRunnableConfig } from "@langchain/langgraph";
      import { z } from "zod";

      const queryDatabase = tool(
        async (input, config: LangGraphRunnableConfig) => {
          config.writer({ data: "Retrieved 0/100 records", type: "progress" }); // (1)!
          // perform query
          config.writer({ data: "Retrieved 100/100 records", type: "progress" }); // (2)!
          return "some-answer";
        },
        {
          name: "query_database",
          description: "Query the database.",
          schema: z.object({
            query: z.string().describe("The query to execute."),
          }),
        }
      );

      const graph = // ... define a graph that uses this tool

      for await (const chunk of await graph.stream(inputs, { streamMode: "custom" })) { // (3)!
        console.log(chunk);
      }
      ```

      1. 使用 writer 发出自定义键值对（例如，进度更新）。
      2. 发出另一个自定义键值对。
      3. 设置 `streamMode: "custom"` 以在流中接收自定义数据。

:::

### 与任何 LLM 配合使用

:::python
您可以使用 `stream_mode="custom"` 从**任何 LLM API** 流式传输数据 — 即使该 API **不**实现 LangChain 聊天模型接口。

这允许您集成原始 LLM 客户端或提供自身流式接口的外部服务，使 LangGraph 能够灵活地进行自定义设置。

```python
from langgraph.config import get_stream_writer

def call_arbitrary_model(state):
    """Example node that calls an arbitrary model and streams the output"""
    # highlight-next-line
    writer = get_stream_writer() # (1)!
    # Assume you have a streaming client that yields chunks
    for chunk in your_custom_streaming_client(state["topic"]): # (2)!
        # highlight-next-line
        writer({"custom_llm_chunk": chunk}) # (3)!
    return {"result": "completed"}

graph = (
    StateGraph(State)
    .add_node(call_arbitrary_model)
    # Add other nodes and edges as needed
    .compile()
)

for chunk in graph.stream(
    {"topic": "cats"},
    # highlight-next-line
    stream_mode="custom", # (4)!
):
    # The chunk will contain the custom data streamed from the llm
    print(chunk)
```

1. 获取流写入器以发送自定义数据。
2. 使用自定义流式客户端生成 LLM token。
3. 使用写入器将自定义数据发送到流。
4. 设置 `stream_mode="custom"` 以在流中接收自定义数据。
   :::

:::js
您可以使用 `streamMode: "custom"` 从**任何 LLM API** 流式传输数据 — 即使该 API **不**实现 LangChain 聊天模型接口。

这允许您集成原始 LLM 客户端或提供自身流式接口的外部服务，使 LangGraph 能够灵活地进行自定义设置。

```typescript
import { LangGraphRunnableConfig } from "@langchain/langgraph";

const callArbitraryModel = async (
  state: any,
  config: LangGraphRunnableConfig
) => {
  // Example node that calls an arbitrary model and streams the output
  // Assume you have a streaming client that yields chunks
  for await (const chunk of yourCustomStreamingClient(state.topic)) {
    // (1)!
    config.writer({ custom_llm_chunk: chunk }); // (2)!
  }
  return { result: "completed" };
};

const graph = new StateGraph(State)
  .addNode("callArbitraryModel", callArbitraryModel)
  // Add other nodes and edges as needed
  .compile();

for await (const chunk of await graph.stream(
  { topic: "cats" },
  { streamMode: "custom" } // (3)!
)) {
  // The chunk will contain the custom data streamed from the llm
  console.log(chunk);
}
```

1. 使用自定义流式客户端生成 LLM token。
2. 使用写入器将自定义数据发送到流。
3. 设置 `streamMode: "custom"` 以在流中接收自定义数据。
   :::

??? example "扩展示例：流式传输任意聊天模型"

      :::python
      ```python
      import operator
      import json

      from typing import TypedDict
      from typing_extensions import Annotated
      from langgraph.graph import StateGraph, START

      from openai import AsyncOpenAI

      openai_client = AsyncOpenAI()
      model_name = "gpt-4o-mini"


      async def stream_tokens(model_name: str, messages: list[dict]):
          response = await openai_client.chat.completions.create(
              messages=messages, model=model_name, stream=True
          )
          role = None
          async for chunk in response:
              delta = chunk.choices[0].delta

              if delta.role is not None:
                  role = delta.role

              if delta.content:
                  yield {"role": role, "content": delta.content}


      # this is our tool
      async def get_items(place: str) -> str:
          """Use this tool to list items one might find in a place you're asked about."""
          writer = get_stream_writer()
          response = ""
          async for msg_chunk in stream_tokens(
              model_name,
              [
                  {
                      "role": "user",
                      "content": (
                          "Can you tell me what kind of items "
                          f"i might find in the following place: '{place}'. "
                          "List at least 3 such items separating them by a comma. "
                          "And include a brief description of each item."
                      ),
                  }
              ],
          ):
              response += msg_chunk["content"]
              writer(msg_chunk)

          return response


      class State(TypedDict):
          messages: Annotated[list[dict], operator.add]


      # this is the tool-calling graph node
      async def call_tool(state: State):
          ai_message = state["messages"][-1]
          tool_call = ai_message["tool_calls"][-1]

          function_name = tool_call["function"]["name"]
          if function_name != "get_items":
              raise ValueError(f"Tool {function_name} not supported")

          function_arguments = tool_call["function"]["arguments"]
          arguments = json.loads(function_arguments)

          function_response = await get_items(**arguments)
          tool_message = {
              "tool_call_id": tool_call["id"],
              "role": "tool",
              "name": function_name,
              "content": function_response,
          }
          return {"messages": [tool_message]}


      graph = (
          StateGraph(State)
          .add_node(call_tool)
          .add_edge(START, "call_tool")
          .compile()
      )
      ```

      Let's invoke the graph with an AI message that includes a tool call:

      ```python
      inputs = {
          "messages": [
              {
                  "content": None,
                  "role": "assistant",
                  "tool_calls": [
                      {
                          "id": "1",
                          "function": {
                              "arguments": '{"place":"bedroom"}',
                              "name": "get_items",
                          },
                          "type": "function",
                      }
                  ],
              }
          ]
      }

      async for chunk in graph.astream(
          inputs,
          stream_mode="custom",
      ):
          print(chunk["content"], end="|", flush=True)
      ```
      :::

      :::js
      ```typescript
      import { StateGraph, START, LangGraphRunnableConfig } from "@langchain/langgraph";
      import { z } from "zod";
      import OpenAI from "openai";

      const openaiClient = new OpenAI();
      const modelName = "gpt-4o-mini";

      async function* streamTokens(modelName: string, messages: any[]) {
        const response = await openaiClient.chat.completions.create({
          messages,
          model: modelName,
          stream: true,
        });

        let role: string | null = null;
        for await (const chunk of response) {
          const delta = chunk.choices[0]?.delta;

          if (delta?.role) {
            role = delta.role;
          }

          if (delta?.content) {
            yield { role, content: delta.content };
          }
        }
      }

      // this is our tool
      const getItems = tool(
        async (input, config: LangGraphRunnableConfig) => {
          let response = "";
          for await (const msgChunk of streamTokens(
            modelName,
            [
              {
                role: "user",
                content: `Can you tell me what kind of items i might find in the following place: '${input.place}'. List at least 3 such items separating them by a comma. And include a brief description of each item.`,
              },
            ]
          )) {
            response += msgChunk.content;
            config.writer?.(msgChunk);
          }
          return response;
        },
        {
          name: "get_items",
          description: "Use this tool to list items one might find in a place you're asked about.",
          schema: z.object({
            place: z.string().describe("The place to look up items for."),
          }),
        }
      );

      const State = z.object({
        messages: z.array(z.any()),
      });

      const graph = new StateGraph(State)
        // this is the tool-calling graph node
        .addNode("callTool", async (state) => {
          const aiMessage = state.messages.at(-1);
          const toolCall = aiMessage.tool_calls?.at(-1);

          const functionName = toolCall?.function?.name;
          if (functionName !== "get_items") {
            throw new Error(`Tool ${functionName} not supported`);
          }

          const functionArguments = toolCall?.function?.arguments;
          const args = JSON.parse(functionArguments);

          const functionResponse = await getItems.invoke(args);
          const toolMessage = {
            tool_call_id: toolCall.id,
            role: "tool",
            name: functionName,
            content: functionResponse,
          };
          return { messages: [toolMessage] };
        })
        .addEdge(START, "callTool")
        .compile();
      ```

      Let's invoke the graph with an AI message that includes a tool call:

      ```typescript
      const inputs = {
        messages: [
          {
            content: null,
            role: "assistant",
            tool_calls: [
              {
                id: "1",
                function: {
                  arguments: '{"place":"bedroom"}',
                  name: "get_items",
                },
                type: "function",
              }
            ],
          }
        ]
      };

      for await (const chunk of await graph.stream(
        inputs,
        { streamMode: "custom" }
      )) {
        console.log(chunk.content + "|");
      }
      ```
      :::

### 为特定的聊天模型禁用流式传输

如果您的应用程序混合了支持流式传输和不支持流式传输的模型，您可能需要显式地为不支持流式传输的模型禁用流式传输。

:::python
在初始化模型时设置 `disable_streaming=True`。

=== "init_chat_model"

      ```python
      from langchain.chat_models import init_chat_model

      model = init_chat_model(
          "anthropic:claude-3-7-sonnet-latest",
          # highlight-next-line
          disable_streaming=True # (1)!
      )
      ```

      1. 设置 `disable_streaming=True` 以禁用聊天模型的流式传输。

=== "chat model interface"

      ```python
      from langchain_openai import ChatOpenAI

      llm = ChatOpenAI(model="o1-preview", disable_streaming=True) # (1)!
      ```

      1. 设置 `disable_streaming=True` 以禁用聊天模型的流式传输。

:::

:::js
在初始化模型时设置 `streaming: false`。

```typescript
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  model: "o1-preview",
  streaming: false, // (1)!
});
```

:::

:::python

### Python < 3.11 的异步 { #async }

在 Python < 3.11 版本中，[asyncio 任务](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task) 不支持 `context` 参数。
这限制了 LangGraph 自动传播上下文的能力，并以两种关键方式影响 LangGraph 的流式传输机制：

1. 您**必须**显式地将 [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) 传递给异步 LLM 调用（例如 `ainvoke()`），因为回调不会自动传播。
2. 您**不能**在异步节点或工具中使用 `get_stream_writer()` — 您必须直接传递 `writer` 参数。

??? example "扩展示例：带有手动配置的异步 LLM 调用"

      ```python
      from typing import TypedDict
      from langgraph.graph import START, StateGraph
      from langchain.chat_models import init_chat_model

      llm = init_chat_model(model="openai:gpt-4o-mini")

      class State(TypedDict):
          topic: str
          joke: str

      async def call_model(state, config): # (1)!
          topic = state["topic"]
          print("Generating joke...")
          joke_response = await llm.ainvoke(
              [{"role": "user", "content": f"Write a joke about {topic}"}],
              # highlight-next-line
              config, # (2)!
          )
          return {"joke": joke_response.content}

      graph = (
          StateGraph(State)
          .add_node(call_model)
          .add_edge(START, "call_model")
          .compile()
      )

      async for chunk, metadata in graph.astream(
          {"topic": "ice cream"},
          # highlight-next-line
          stream_mode="messages", # (3)!
      ):
          if chunk.content:
              print(chunk.content, end="|", flush=True)
      ```

      1. 在异步节点函数中接受 `config` 作为参数。
      2. 将 `config` 传递给 `llm.ainvoke()` 以确保正确的上下文传播。
      3. 设置 `stream_mode="messages"` 以流式传输 LLM token。

??? example "扩展示例：带有流写入器的异步自定义流式传输"

      ```python
      from typing import TypedDict
      from langgraph.types import StreamWriter

      class State(TypedDict):
            topic: str
            joke: str

      # highlight-next-line
      async def generate_joke(state: State, writer: StreamWriter): # (1)!
            writer({"custom_key": "Streaming custom data while generating a joke"})
            return {"joke": f"This is a joke about {state['topic']}"}

      graph = (
            StateGraph(State)
            .add_node(generate_joke)
            .add_edge(START, "generate_joke")
            .compile()
      )

      async for chunk in graph.astream(
            {"topic": "ice cream"},
            # highlight-next-line
            stream_mode="custom", # (2)!
      ):
            print(chunk)
      ```

      1. 在异步节点或工具的函数签名中添加 `writer` 作为参数。LangGraph 将自动将流写入器传递给该函数。
      2. 设置 `stream_mode="custom"` 以在流中接收自定义数据。

:::