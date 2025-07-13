# 流式 API

[LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 允许你从 LangGraph API 服务器流式传输输出。

!!! note

    LangGraph SDK 和 LangGraph Server 是 [LangGraph Platform](../../concepts/langgraph_platform.md) 的一部分。

## 基本用法

基本用法示例：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    # 使用名为 "agent" 的已部署图
    assistant_id = "agent"

    # 创建一个线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 创建一个流式运行
    # highlight-next-line
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode="updates"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // 使用名为 "agent" 的已部署图
    const assistantID = "agent";

    // 创建一个线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 创建一个流式运行
    // highlight-next-line
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    创建线程：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    创建流式运行：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <inputs>,
      \"stream_mode\": \"updates\"
    }"
    ```

??? example "扩展示例：流式更新"

    这是你可以在 LangGraph API 服务器中运行的示例图。
    更多详情请参阅 [LangGraph Platform 快速入门](../quick_start.md)。

    ```python
    # graph.py
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

    一旦有一个正在运行的 LangGraph API 服务器，你就可以使用
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 进行交互

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # 使用名为 "agent" 的已部署图
        assistant_id = "agent"

        # 创建一个线程
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # 创建一个流式运行
        # highlight-next-line
        async for chunk in client.runs.stream(  # (1)!
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="updates"  # (2)!
        ):
            print(chunk.data)
        ```

        1. `client.runs.stream()` 方法返回一个生成流式输出的迭代器。
        2. 设置 `stream_mode="updates"` 以仅流式传输每个节点后的图状态更新。也支持其他流式模式。详情请参阅[支持的流式模式](#supported-stream-modes)。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // 使用名为 "agent" 的已部署图
        const assistantID = "agent";

        // 创建一个线程
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // 创建一个流式运行
        // highlight-next-line
        const streamResponse = client.runs.stream(  // (1)!
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            // highlight-next-line
            streamMode: "updates"  // (2)!
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

        1. `client.runs.stream()` 方法返回一个生成流式输出的迭代器。
        2. 设置 `streamMode: "updates"` 以仅流式传输每个节点后的图状态更新。也支持其他流式模式。详情请参阅[支持的流式模式](#supported-stream-modes)。

    === "cURL"

        创建线程：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        创建流式运行：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"updates\"
        }"
        ```

    ```output
    {'run_id': '1f02c2b3-3cef-68de-b720-eec2a4a8e920', 'attempt': 1}
    {'refine_topic': {'topic': 'ice cream and cats'}}
    {'generate_joke': {'joke': 'This is a joke about ice cream and cats'}}
    ```

### 支持的流式模式

| 模式                             | 描述                                                                                                                                            | LangGraph 库方法                                                                                 |
| ---------------------------------|-------------------------------------------------------------------------------------------------------------------------------------------------|-------------------------------------------------------------------------------------------------|
| [`values`](#stream-graph-state)  | 流式传输每个 [超级步骤](../../concepts/low_level.md#graphs) 后的完整图状态。                                                                        | 使用 [`stream_mode="values"`](../../how-tos/streaming.md#stream-graph-state) 的 `.stream()` / `.astream()` |
| [`updates`](#stream-graph-state) | 流式传输每个步骤后的状态更新。如果同一步骤进行了多次更新（例如，运行了多个节点），则这些更新会单独流式传输。                                                     | 使用 [`stream_mode="updates"`](../../how-tos/streaming.md#stream-graph-state) 的 `.stream()` / `.astream()` |
| [`messages-tuple`](messeges)    | 流式传输 LLM token 和元数据，用于调用 LLM 的图节点（对聊天应用很有用）。                                                                           | 使用 [`stream_mode="messages"`](../../how-tos/streaming.md#messages) 的 `.stream()` / `.astream()`          |
| [`debug`](#debug)                | 流式传输图执行过程中尽可能多的信息。                                                                                                          | 使用 [`stream_mode="debug"`](../../how-tos/streaming.md#stream-graph-state) 的 `.stream()` / `.astream()`   |
| [`custom`](#stream-custom-data)  | 流式传输图内的自定义数据                                                                                                                      | 使用 [`stream_mode="custom"`](../../how-tos/streaming.md#stream-custom-data) 的 `.stream()` / `.astream()`  |
| [`events`](#stream-events)       | 流式传输所有事件（包括图的状态）；主要用于迁移大型 LCEL 应用。                                                                                | `.astream_events()`                                                                             |

### 流式传输多种模式

你可以将 `stream_mode` 参数传递为一个列表，以同时流式传输多种模式。

流式输出将是 `(mode, chunk)` 元组，其中 `mode` 是流式模式的名称，`chunk` 是该模式流式传输的数据。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input=inputs,
        stream_mode=["updates", "custom"]
    ):
        print(chunk)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input,
        streamMode: ["updates", "custom"]
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"input\": <inputs>,
       \"stream_mode\": [
         \"updates\"
         \"custom\"
       ]
     }"
    ```

## 流式传输图状态

使用 `updates` 和 `values` 流式模式来流式传输图在执行过程中的状态。

* `updates` 流式传输每个步骤后状态的**更新**。
* `values` 流式传输每个步骤后状态的**完整值**。

??? example "示例图"

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

!!! note "有状态运行"

    下面的示例假定你希望将流式运行的输出**持久化**到 [checkpointer](../../concepts/persistence.md) 数据库中，并且已经创建了一个线程。要创建线程：

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # 使用名为 "agent" 的已部署图
        assistant_id = "agent"
        # 创建一个线程
        thread = await client.threads.create()
        thread_id = thread["thread_id"]
        ```

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // 使用名为 "agent" 的已部署图
        const assistantID = "agent";
        // 创建一个线程
        const thread = await client.threads.create();
        const threadID = thread["thread_id"]
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

    如果你不需要持久化运行的输出，可以在流式传输时将 `thread_id` 替换为 `None`。

=== "updates"

    使用此选项可仅流式传输每个步骤后节点返回的**状态更新**。流式输出包括节点名称和更新内容。

    === "Python"

        ```python
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="updates"
        ):
            print(chunk.data)
        ```

    === "JavaScript"

        ```js
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            # highlight-next-line
            streamMode: "updates"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"updates\"
        }"
        ```

===  "values"

    使用此选项可流式传输每个步骤后的图**完整状态**。

    === "Python"

        ```python
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"topic": "ice cream"},
            # highlight-next-line
            stream_mode="values"
        ):
            print(chunk.data)
        ```

    === "JavaScript"

        ```js
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { topic: "ice cream" },
            # highlight-next-line
            streamMode: "values"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk.data);
        }
        ```

    === "cURL"

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"topic\": \"ice cream\"},
          \"stream_mode\": \"values\"
        }"
        ```

## 子图

要将 [子图](../../concepts/subgraphs.md) 的输出包含在流式输出中，你可以在父图的 `.stream()` 方法中设置 `subgraphs=True`。这将流式传输父图和任何子图的输出。

```python
for chunk in client.runs.stream(
    thread_id,
    assistant_id,
    input={"foo": "foo"},
    # highlight-next-line
    stream_subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. 设置 `stream_subgraphs=True` 以流式传输子图的输出。

??? example "扩展示例：从子图流式传输"

    这是你可以在 LangGraph API 服务器中运行的示例图。
    更多详情请参阅 [LangGraph Platform 快速入门](../quick_start.md)。

    ```python
    # graph.py
    from langgraph.graph import START, StateGraph
    from typing import TypedDict

    # 定义子图
    class SubgraphState(TypedDict):
        foo: str  # 注意这个键与父图状态共享
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
    ```

    一旦有一个正在运行的 LangGraph API 服务器，你就可以使用
    [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 进行交互

    === "Python"

        ```python
        from langgraph_sdk import get_client
        client = get_client(url=<DEPLOYMENT_URL>)

        # 使用名为 "agent" 的已部署图
        assistant_id = "agent"

        # 创建一个线程
        thread = await client.threads.create()
        thread_id = thread["thread_id"]
    
        async for chunk in client.runs.stream(
            thread_id,
            assistant_id,
            input={"foo": "foo"},
            # highlight-next-line
            stream_subgraphs=True, # (1)!
            stream_mode="updates",
        ):
            print(chunk)
        ```
        
        1. 设置 `stream_subgraphs=True` 以流式传输子图的输出。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // 使用名为 "agent" 的已部署图
        const assistantID = "agent";

        // 创建一个线程
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // 创建一个流式运行
        const streamResponse = client.runs.stream(
          threadID,
          assistantID,
          {
            input: { foo: "foo" },
            # highlight-next-line
            streamSubgraphs: true,  # (1)!
            streamMode: "updates"
          }
        );
        for await (const chunk of streamResponse) {
          console.log(chunk);
        }
        ```

        1. 设置 `streamSubgraphs: true` 以流式传输子图的输出。

    === "cURL"

        创建线程：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        创建流式运行：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"foo\": \"foo\"},
          \"stream_subgraphs\": true,
          \"stream_mode\": [
            \"updates\"
          ]
        }"
        ```

    **请注意**，我们不仅收到了节点更新，还收到了命名空间，它们告诉我们正在从哪个图（或子图）流式传输。

## 调试 {#debug}

使用 `debug` 流式模式来流式传输图执行过程中尽可能多的信息。流式输出包括节点名称和完整状态。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="debug"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        # highlight-next-line
        streamMode: "debug"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"debug\"
    }"
    ```

## LLM token {#messages}

使用 `messages-tuple` 流式模式来逐 token 流式传输大型语言模型（LLM）的输出，无论是在图的任何部分，包括节点、工具、子图或任务。

从 [`messages-tuple` 模式](#supported-stream-modes) 流式传输的输出是一个 `(message_chunk, metadata)` 元组，其中：

- `message_chunk`：来自 LLM 的 token 或消息片段。
- `metadata`：包含关于图节点和 LLM 调用详细信息的字典。
 
??? example "示例图"

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
        """调用 LLM 根据主题生成笑话"""
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
    ```

    1. 请注意，即使 LLM 使用 `.invoke` 而不是 `.stream` 调用，也会发出消息事件。

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="messages-tuple",
    ):
        if chunk.event != "messages":
            continue

        message_chunk, metadata = chunk.data  # (1)!
        if message_chunk["content"]:
            print(message_chunk["content"], end="|", flush=True)
    ```

    1. "messages-tuple" 流式模式返回一个 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式传输的 token，而 `metadata` 是一个包含 LLM 调用所在图节点和其他信息的字典。

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        # highlight-next-line
        streamMode: "messages-tuple"
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.event !== "messages") {
        continue;
      }
      console.log(chunk.data[0]["content"]);  # (1)!
    }
    ```

    1. "messages-tuple" 流式模式返回一个 `(message_chunk, metadata)` 元组的迭代器，其中 `message_chunk` 是 LLM 流式传输的 token，而 `metadata` 是一个包含 LLM 调用所在图节点和其他信息的字典。

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"messages-tuple\"
    }"
    ```

### 过滤 LLM token

* 要按 LLM 调用过滤流式 token，你可以将 `tags` 与 LLM 调用关联起来（请参阅[按 LLM 调用过滤](../../how-tos/streaming.md#filter-by-llm-invocation)）。
* 要仅流式传输特定节点的 token，请使用 `stream_mode="messages"` 并根据流式元数据中的 `langgraph_node` 字段进行过滤（请参阅[按节点过滤](../../how-tos/streaming.md#filter-by-node)）。

## 流式传输自定义数据

要发送**用户定义的自定义数据**：

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"query": "example"},
        # highlight-next-line
        stream_mode="custom"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { query: "example" },
        # highlight-next-line
        streamMode: "custom"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"query\": \"example\"},
      \"stream_mode\": \"custom\"
    }"
    ```

## 流式传输事件

要流式传输所有事件，包括图的状态：

=== "Python"

    ```python
    async for chunk in client.runs.stream(
        thread_id,
        assistant_id,
        input={"topic": "ice cream"},
        # highlight-next-line
        stream_mode="events"
    ):
        print(chunk.data)
    ```

=== "JavaScript"

    ```js
    const streamResponse = client.runs.stream(
      threadID,
      assistantID,
      {
        input: { topic: "ice cream" },
        # highlight-next-line
        streamMode: "events"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"topic\": \"ice cream\"},
      \"stream_mode\": \"events\"
    }"
    ```

## 无状态运行

如果你不想将流式运行的输出**持久化**到 [checkpointer](../../concepts/persistence.md) 数据库中，则可以在不创建线程的情况下创建无状态运行：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    async for chunk in client.runs.stream(
        # highlight-next-line
        None,  # (1)!
        assistant_id,
        input=inputs,
        stream_mode="updates"
    ):
        print(chunk.data)
    ```

    1. 这里我们传递的是 `None` 而不是 `thread_id` UUID。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // 创建一个流式运行
    # highlight-next-line
    const streamResponse = client.runs.stream(
      # highlight-next-line
      null,  # (1)!
      assistantID,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      console.log(chunk.data);
    }
    ```

    1. 这里我们传递的是 `null` 而不是 `thread_id` UUID。

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/runs/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <inputs>,
      \"stream_mode\": \"updates\"
    }"
    ```

## 加入并流式传输

LangGraph Platform 允许你加入一个活动的 [后台运行](../how-tos/background_run.md) 并从中流式传输输出。为此，你可以使用 [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 的 `client.runs.join_stream` 方法：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>, api_key=<API_KEY>)

    # highlight-next-line
    async for chunk in client.runs.join_stream(
        thread_id,
        # highlight-next-line
        run_id,  # (1)!
    ):
        print(chunk)
    ```

    1. 这是你想加入的现有运行的 `run_id`。


=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL>, apiKey: <API_KEY> });

    // highlight-next-line
    const streamResponse = client.runs.joinStream(
      threadID,
      # highlight-next-line
      runId  # (1)!
    );
    for await (const chunk of streamResponse) {
      console.log(chunk);
    }
    ```

    1. 这是你想加入的现有运行的 `run_id`。

=== "cURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/<RUN_ID>/stream \
    --header 'Content-Type: application/json' \
    --header 'x-api-key: <API_KEY>'
    ```

!!! warning "输出未缓冲"

    当你使用 `.join_stream` 时，输出不会被缓冲，因此在加入之前产生的任何输出都将无法接收。

## API 参考

有关 API 的用法和实现，请参阅 [API 参考](../reference/api/api_ref.html#tag/thread-runs/POST/threads/{thread_id}/runs/stream)。