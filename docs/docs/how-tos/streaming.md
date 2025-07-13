# 流式输出

您可以从 LangGraph 代理或工作流程中[流式传输输出](../concepts/streaming.md)。

## 支持的流模式

将以下一种或多种流模式作为列表传递给 [`stream()`](langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream()`](langgraph.graph.state.CompiledStateGraph.astream) 方法：

| 模式 | 描述 |
|---|---|
| `values` | 在图的每一步之后流式传输状态的完整值。 |
| `updates` | 在图的每一步之后流式传输状态的更新。如果在同一步骤中进行了多次更新（例如，运行了多个节点），则这些更新会单独流式传输。 |
| `custom` | 从图节点内部流式传输自定义数据。 |
| `messages` | 从调用了 LLM 的任何图节点流式传输 2 元组（LLM token，元数据）。 |
| `debug` | 在图执行的整个过程中尽可能多地流式传输信息。 |

## 从代理流式传输

### 代理进度

要流式传输代理进度，请使用 [`stream()`](langgraph.graph.state.CompiledStateGraph.stream) 或 [`astream()`](langgraph.graph.state.CompiledStateGraph.astream) 方法并设置 `stream_mode="updates"`。这将为每次代理步骤发出一个事件。

例如，如果您有一个调用一次工具的代理，您应该会看到以下更新：

* **LLM 节点**：包含工具调用请求的 AI 消息
* **工具节点**：包含执行结果的工具消息
* **LLM 节点**：最终的 AI 回复

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

### LLM token

要以 token 形式流式传输 LLM 生成的内容，请使用 `stream_mode="messages"`：

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

### 工具更新

要流式传输工具执行过程中的更新，您可以使用 [get_stream_writer](langgraph.config.get_stream_writer)。

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
    如果您在工具中添加了 `get_stream_writer`，您将无法在 LangGraph 执行上下文之外调用该工具。

### 流式传输多种模式

您可以通过将流模式指定为列表来一次流式传输多种模式：`stream_mode=["updates", "messages", "custom"]`：

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

### 禁用流式传输

在某些应用程序中，您可能需要为给定的模型禁用单个 token 的流式传输。这在[多代理](../agents/multi-agent.md)系统中很有用，可以控制哪些代理流式传输其输出。

请参阅[模型](../agents/models.md#disable-streaming)指南了解如何禁用流式传输。

## 从工作流流式传输

### 基本用法示例

LangGraph 图公开了 [`.stream()`](langgraph.pregel.Pregel.stream)（同步）和 [`.astream()`](langgraph.pregel.Pregel.astream)（异步）方法，以产生流式输出作为迭代器。

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

??? 示例 "扩展示例：流式传输更新"

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

      1. `stream()` 方法返回一个产生流式输出的迭代器。
      2. 设置 `stream_mode="updates"` 以仅流式传输每个节点后的图状态更新。也支持其他流模式。有关详细信息，请参阅[支持的流模式](#supported-stream-modes)。

      ```output
      {'refine_topic': {'topic': 'ice cream and cats'}}
      {'generate_joke': {'joke': 'This is a joke about ice cream and cats'}}
      ```

### 流式传输多种模式

您可以将列表作为 `stream_mode` 参数传递，以一次流式传输多种模式。

流式输出将是 `(mode, chunk)` 的元组，其中 `mode` 是流模式的名称，`chunk` 是该模式流式传输的数据。

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

### 流式传输图状态

使用 `updates` 和 `values` 流模式来流式传输图执行时的状态。

* `updates` 在每一步之后流式传输状态的**更新**。
* `values` 在每一步之后流式传输状态的**完整值**。

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

=== "updates"

    使用此选项仅流式传输每个节点返回后的**状态更新**。流式输出包括节点名称和更新。


    ```python
    for chunk in graph.stream(
        {"topic": "ice cream"},
        # highlight-next-line
        stream_mode="updates",
    ):
        print(chunk)
    ```

=== "values"

    使用此选项在每一步之后流式传输图的**完整状态**。

    ```python
    for chunk in graph.stream(
        {"topic": "ice cream"},
        # highlight-next-line
        stream_mode="values",
    ):
        print(chunk)
    ```

### 流式传输子图输出

要将[子图](../concepts/subgraphs.md)的输出包含在流式输出中，您可以在父图的 `.stream()` 方法中设置 `subgraphs=True`。这将流式传输父图和任何子图的输出。

输出将作为 `(namespace, data)` 的元组进行流式传输，其中 `namespace` 是一个包含调用子图的节点路径的元组，例如 `("parent_node:<task_id>", "child_node:<task_id>")`。

```python
for chunk in graph.stream(
    {"foo": "foo"},
    # highlight-next-line
    subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. 设置 `subgraphs=True` 以从子图中流式传输输出。

??? 示例 "扩展示例：从子图流式传输"

      ```python
      from langgraph.graph import START, StateGraph
      from typing import TypedDict

      # 定义子图
      class SubgraphState(TypedDict):
          foo: str  # 注意此键与父图状态共享
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

      # 定义父图
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
      
      1. 设置 `subgraphs=True` 以从子图中流式传输输出。

      ```
      ((), {'node_1': {'foo': 'hi! foo'}})
      (('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',), {'subgraph_node_1': {'bar': 'bar'}})
      (('node_2:dfddc4ba-c3c5-6887-5012-a243b5b377c2',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
      ((), {'node_2': {'foo': 'hi! foobar'}})
      ```

      请注意，我们不仅收到了节点更新，还收到了命名空间，它们告诉我们正在从哪个图（或子图）进行流式传输。

### 调试 {#debug}

使用 `debug` 流模式在图的执行过程中尽可能多地流式传输信息。流式输出包括节点名称和整个状态。

```python
for chunk in graph.stream(
    {"topic": "ice cream"},
    # highlight-next-line
    stream_mode="debug",
):
    print(chunk)
```

### LLM token {#messages}

使用 `messages` 流模式以**逐 token** 的方式流式传输大型语言模型 (LLM) 的输出，无论是在图的任何部分，包括节点、工具、子图或任务。

[`messages`模式](#supported-stream-modes)的流式输出是一个 `(message_chunk, metadata)` 元组，其中：

- `message_chunk`：来自 LLM 的 token 或消息片段。
- `metadata`：一个包含图节点和 LLM 调用详细信息的字典。

> 如果您的 LLM 未作为 LangChain 集成提供，则可以使用 `custom` 模式进行流式传输。有关详细信息，请参阅[与任何 LLM 一起使用](#use-with-any-llm)。
 
!!! 警告 "Python < 3.11 的 Async 需要手动配置"

    在使用 Python < 3.11 并运行异步代码时，您必须显式地将 `RunnableConfig` 传递给 `ainvoke()` 以启用正确的流式传输。有关详细信息，请参阅[Python < 3.11 的 Async](#async) 或升级到 Python 3.11+。

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

1. 请注意，即使 LLM 使用 `.invoke` 而不是 `.stream` 运行，也会发出消息事件。
2. "messages" 流模式返回一个 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式的 token，`metadata` 是包含 LLM 调用所在的图节点信息和其他信息的字典。

#### 按 LLM 调用过滤

您可以为 LLM 调用关联 `tags`，以按 LLM 调用过滤流式传输的 token。

```python
from langchain.chat_models import init_chat_model

joke_model = init_chat_model(model="openai:gpt-4o-mini", tags=['joke']) # (1)!
poem_model = init_chat_model(model="openai:gpt-4o-mini", tags=['poem']) # (2)!

graph = ... # 定义一个使用这些 LLM 的图

async for msg, metadata in graph.astream(  # (3)!
    {"topic": "cats"},
    # highlight-next-line
    stream_mode="messages",
):
    if metadata["tags"] == ["joke"]: # (4)!
        print(msg.content, end="|", flush=True)
```

1. joke_model 被标记为 "joke"。
2. poem_model 被标记为 "poem"。
3. `stream_mode` 设置为 "messages" 以流式传输 LLM token。 `metadata` 包含 LLM 调用信息，包括标签。
4. 通过元数据中的 `tags` 字段过滤流式传输的 token，以仅包含带有 "joke" 标签的 LLM 调用的 token。

??? 示例 "扩展示例：按标签过滤"

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
            # 注意：显式传递 config 对于 Python < 3.11 是必需的
            # 因为上下文变量支持在此之前并未添加：https://docs.python.org/3/library/asyncio-task.html#creating-tasks
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

      1. `joke_model` 被标记为 "joke"。
      2. `poem_model` 被标记为 "poem"。
      3. 显式传递 `config` 以确保正确传播上下文变量。这对于 Python < 3.11 使用异步代码是必需的。有关更多详细信息，请参阅[异步部分](#async)。
      4. `stream_mode` 设置为 "messages" 以流式传输 LLM token。 `metadata` 包含 LLM 调用信息，包括标签。

#### 按节点过滤

要仅从特定节点流式传输 token，请使用 `stream_mode="messages"` 并按流式传输元数据中的 `langgraph_node` 字段进行过滤：

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

1. "messages" 流模式返回一个 `(message_chunk, metadata)` 元组，其中 `message_chunk` 是 LLM 流式的 token，`metadata` 是包含 LLM 调用所在的图节点信息和其他信息的字典。
2. 通过元数据中的 `langgraph_node` 字段过滤流式传输的 token，以仅包含来自 `write_poem` 节点的 token。

??? 示例 "扩展示例：从特定节点流式传输 LLM token"

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
            # concurrently write both the joke and the poem
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

      1. "messages" 流模式返回一个 `(message_chunk, metadata)` 元组，其中 `message_chunk` 是 LLM 流式的 token，`metadata` 是包含 LLM 调用所在的图节点信息和其他信息的字典。
      2. 通过元数据中的 `langgraph_node` 字段过滤流式传输的 token，以仅包含来自 `write_poem` 节点的 token。

### 流式传输自定义数据

要从 LangGraph 节点或工具中发送**用户定义的自定义数据**，请遵循以下步骤：

1. 使用 `get_stream_writer()` 访问流式写入器并发出自定义数据。
2. 在调用 `.stream()` 或 `.astream()` 时设置 `stream_mode="custom"` 以在流中获取自定义数据。您可以组合多种模式（例如 `["updates", "custom"]`），但至少一种模式必须是 `"custom"`。

!!! 警告 "Python < 3.11 的 Async 中没有 `get_stream_writer()`"

    在 Python < 3.11 上运行的异步代码中，`get_stream_writer()` 将无法正常工作。
    相反，请在节点或工具中添加 `writer` 参数，并手动传递它。
    有关用法示例，请参阅[Python < 3.11 的 Async](#async)。


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

      1. 获取流式写入器以发送自定义数据。
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


      graph = ... # 定义一个使用此工具的图

      for chunk in graph.stream(inputs, stream_mode="custom"): # (4)!
          print(chunk)
      ```

      1. 访问流式写入器以发送自定义数据。
      2. 发出自定义键值对（例如，进度更新）。
      3. 发出另一个自定义键值对。
      4. 设置 `stream_mode="custom"` 以在流中接收自定义数据。

### 与任何 LLM 一起使用

您可以使用 `stream_mode="custom"` 来流式传输**任何 LLM API** 的数据 — 即使该 API **没有**实现 LangChain 聊天模型接口。

这使您可以集成原始 LLM 客户端或提供自己流式接口的外部服务，从而使 LangGraph 在自定义设置中具有高度灵活性。

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

1. 获取流式写入器以发送自定义数据。
2. 使用自定义流式客户端生成 LLM token。
3. 使用写入器将自定义数据发送到流。
4. 设置 `stream_mode="custom"` 以在流中接收自定义数据。


??? 示例 "扩展示例：流式传输任意聊天模型"
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

      let's invoke the graph with an AI message that includes a tool call:

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

### 为特定聊天模型禁用流式传输

如果您的应用程序混合了支持流式传输和不支持流式传输的模型，您可能需要显式为不支持的模型禁用流式传输。

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

### Python < 3.11 的 Async { #async }

在 Python 版本 < 3.11 中，[asyncio 任务](https://docs.python.org/3/library/asyncio-task.html#asyncio.create_task)不支持 `context` 参数。
这限制了 LangGraph 自动传播上下文的能力，并以两种关键方式影响 LangGraph 的流式传输机制：

1. 您**必须**将 [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) 显式传递给异步 LLM 调用（例如 `ainvoke()`），因为回调不会自动传播。
2. 您**不能**在异步节点或工具中使用 `get_stream_writer()` — 您必须直接传递 `writer` 参数。

??? 示例 "扩展示例：使用手动配置的异步 LLM 调用"

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

??? 示例 "扩展示例：带有流式写入器的异步自定义流式传输"

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

      1. 在异步节点或工具的函数签名中添加 `writer` 作为参数。LangGraph 将自动将流式写入器传递给函数。
      2. 设置 `stream_mode="custom"` 以在流中接收自定义数据。