# 使用 RemoteGraph 交互部署

!!! info "先决条件"

    - [LangGraph Platform](../concepts/langgraph_platform.md)
    - [LangGraph Server](../concepts/langgraph_server.md)

`RemoteGraph` 是一个接口，它允许您像与常规本地定义的 LangGraph 图（例如 `CompiledGraph`）一样与之交互。本指南将展示如何初始化 `RemoteGraph` 并与之进行交互。

## 初始化图

:::python

在初始化 `RemoteGraph` 时，您必须始终指定：

- `name`：您要交互的图的名称。这与您在部署的 `langgraph.json` 配置文件中使用的图名称相同。
- `api_key`：一个有效的 LangSmith API 密钥。可以将其设置为环境变量 (`LANGSMITH_API_KEY`)，也可以通过 `api_key` 参数直接传递。如果 `LangGraphClient` / `SyncLangGraphClient` 是使用 `api_key` 参数初始化的，也可以通过 `client` / `sync_client` 参数提供 API 密钥。

此外，您还必须提供以下项之一：

- `url`：要交互的部署的 URL。如果您传递 `url` 参数，同步和异步客户端都将使用提供的 URL、标头（如果提供）和默认配置值（例如，超时等）进行创建。
- `client`：一个 `LangGraphClient` 实例，用于异步与部署进行交互（例如，使用 `.astream()`、`.ainvoke()`、`.aget_state()`、`.aupdate_state()` 等）。
- `sync_client`：一个 `SyncLangGraphClient` 实例，用于同步与部署进行交互（例如，使用 `.stream()`、`.invoke()`、`.get_state()`、`.update_state()` 等）。

!!! Note

    如果您同时传递 `client` 或 `sync_client` 以及 `url` 参数，它们将优先于 `url` 参数。如果未提供 `client` / `sync_client` / `url` 参数中的任何一个，`RemoteGraph` 将在运行时引发 `ValueError`。

:::

:::js

在初始化 `RemoteGraph` 时，您必须始终指定：

- `name`：您要交互的图的名称。这与您在部署的 `langgraph.json` 配置文件中使用的图名称相同。
- `apiKey`：一个有效的 LangSmith API 密钥。可以将其设置为环境变量 (`LANGSMITH_API_KEY`)，也可以通过 `apiKey` 参数直接传递。如果 `LangGraphClient` 是使用 `apiKey` 参数初始化的，也可以通过 `client` 参数提供 API 密钥。

此外，您还必须提供以下项之一：

- `url`：要交互的部署的 URL。如果您传递 `url` 参数，同步和异步客户端都将使用提供的 URL、标头（如果提供）和默认配置值（例如，超时等）进行创建。
- `client`：一个 `LangGraphClient` 实例，用于异步与部署进行交互。

:::

### 使用 URL

:::python

```python
from langgraph.pregel.remote import RemoteGraph

url = <DEPLOYMENT_URL>
graph_name = "agent"
remote_graph = RemoteGraph(graph_name, url=url)
```

:::

:::js

```ts
import { RemoteGraph } from "@langchain/langgraph/remote";

const url = `<DEPLOYMENT_URL>`;
const graphName = "agent";
const remoteGraph = new RemoteGraph({ graphId: graphName, url });
```

:::

### 使用客户端

:::python

```python
from langgraph_sdk import get_client, get_sync_client
from langgraph.pregel.remote import RemoteGraph

url = <DEPLOYMENT_URL>
graph_name = "agent"
client = get_client(url=url)
sync_client = get_sync_client(url=url)
remote_graph = RemoteGraph(graph_name, client=client, sync_client=sync_client)
```

:::

:::js

```ts
import { Client } from "@langchain/langgraph-sdk";
import { RemoteGraph } from "@langchain/langgraph/remote";

const client = new Client({ apiUrl: `<DEPLOYMENT_URL>` });
const graphName = "agent";
const remoteGraph = new RemoteGraph({ graphId: graphName, client });
```

:::

## 调用图

:::python
由于 `RemoteGraph` 是一个实现了与 `CompiledGraph` 相同方法的 `Runnable`，因此您可以像平常使用已编译的图一样与它交互，即通过调用 `.invoke()`、`.stream()`、`.get_state()`、`.update_state()` 等（以及它们的异步对应方法）。

### 异步调用

!!! Note

    要异步使用图，您必须在初始化 `RemoteGraph` 时提供 `url` 或 `client`。

```python
# 调用图
result = await remote_graph.ainvoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
})

# 从图中流式输出
async for chunk in remote_graph.astream({
    "messages": [{"role": "user", "content": "what's the weather in la"}]
}):
    print(chunk)
```

### 同步调用

!!! Note

    要同步使用图，您必须在初始化 `RemoteGraph` 时提供 `url` 或 `sync_client`。

```python
# 调用图
result = remote_graph.invoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
})

# 从图中流式输出
for chunk in remote_graph.stream({
    "messages": [{"role": "user", "content": "what's the weather in la"}]
}):
    print(chunk)
```

:::

:::js
由于 `RemoteGraph` 是一个实现了与 `CompiledGraph` 相同方法的 `Runnable`，因此您可以像平常使用已编译的图一样与它交互，即通过调用 `.invoke()`、`.stream()`、`.getState()`、`.updateState()` 等。

```ts
// 调用图
const result = await remoteGraph.invoke({
    messages: [{role: "user", content: "what's the weather in sf"}]
})

// 从图中流式输出
for await (const chunk of await remoteGraph.stream({
    messages: [{role: "user", content: "what's the weather in la"}]
})):
    console.log(chunk)
```

:::

## 线程级持久化

默认情况下，图的运行（即 `.invoke()` 或 `.stream()` 调用）是无状态的——图的检查点和最终状态不会被持久化。如果您想持久化图运行的输出（例如，启用人工干预功能），您可以创建一个线程并通过 `config` 参数提供线程 ID，就像使用常规已编译图一样：

:::python

```python
from langgraph_sdk import get_sync_client
url = <DEPLOYMENT_URL>
graph_name = "agent"
sync_client = get_sync_client(url=url)
remote_graph = RemoteGraph(graph_name, url=url)

# 创建一个线程（或使用现有线程）
thread = sync_client.threads.create()

# 使用线程配置调用图
config = {"configurable": {"thread_id": thread["thread_id"]}}
result = remote_graph.invoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
}, config=config)

# 验证状态是否已持久化到线程
thread_state = remote_graph.get_state(config)
print(thread_state)
```

:::

:::js

```ts
import { Client } from "@langchain/langgraph-sdk";
import { RemoteGraph } from "@langchain/langgraph/remote";

const url = `<DEPLOYMENT_URL>`;
const graphName = "agent";
const client = new Client({ apiUrl: url });
const remoteGraph = new RemoteGraph({ graphId: graphName, url });

// 创建一个线程（或使用现有线程）
const thread = await client.threads.create();

// 使用线程配置调用图
const config = { configurable: { thread_id: thread.thread_id } };
const result = await remoteGraph.invoke(
  {
    messages: [{ role: "user", content: "what's the weather in sf" }],
  },
  config
);

// 验证状态是否已持久化到线程
const threadState = await remoteGraph.getState(config);
console.log(threadState);
```

:::

## 用作子图

!!! Note

    如果您需要将 `checkpointer` 用于包含 `RemoteGraph` 子图节点的图，请确保使用 UUID 作为线程 ID。

由于 `RemoteGraph` 的行为与常规 `CompiledGraph` 相同，因此它也可以在另一个图中用作子图。例如：

:::python

```python
from langgraph_sdk import get_sync_client
from langgraph.graph import StateGraph, MessagesState, START
from typing import TypedDict

url = <DEPLOYMENT_URL>
graph_name = "agent"
remote_graph = RemoteGraph(graph_name, url=url)

# 定义父图
builder = StateGraph(MessagesState)
# 直接将远程图添加为节点
builder.add_node("child", remote_graph)
builder.add_edge(START, "child")
graph = builder.compile()

# 调用父图
result = graph.invoke({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
})
print(result)

# 从父图和子图流式输出
for chunk in graph.stream({
    "messages": [{"role": "user", "content": "what's the weather in sf"}]
}, subgraphs=True):
    print(chunk)
```

:::

:::js

```ts
import { MessagesAnnotation, StateGraph, START } from "@langchain/langgraph";
import { RemoteGraph } from "@langchain/langgraph/remote";

const url = `<DEPLOYMENT_URL>`;
const graphName = "agent";
const remoteGraph = new RemoteGraph({ graphId: graphName, url });

// 定义父图并将远程图直接添加为节点
const graph = new StateGraph(MessagesAnnotation)
  .addNode("child", remoteGraph)
  .addEdge(START, "child")
  .compile();

// 调用父图
const result = await graph.invoke({
  messages: [{ role: "user", content: "what's the weather in sf" }],
});
console.log(result);

// 从父图和子图流式输出
for await (const chunk of await graph.stream(
  {
    messages: [{ role: "user", content: "what's the weather in la" }],
  },
  { subgraphs: true }
)) {
  console.log(chunk);
}
```

:::