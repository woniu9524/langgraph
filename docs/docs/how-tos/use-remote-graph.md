# 如何与部署进行交互

!!! info "先决条件"
    - [LangGraph Platform](../concepts/langgraph_platform.md)
    - [LangGraph Server](../concepts/langgraph_server.md)

`RemoteGraph` 是一个接口，允许您与您的 LangGraph Platform 部署进行交互，就像与普通、本地定义的 LangGraph 图（例如 `CompiledGraph`）一样。本指南将展示如何初始化 `RemoteGraph` 并与之交互。

## 初始化图表

初始化 `RemoteGraph` 时，您必须始终指定：

- `name`: 您要交互的图表的名称。这与您在部署的 `langgraph.json` 配置文件中使用的图表名称相同。
- `api_key`: 一个有效的 LangSmith API 密钥。可以设置为环境变量 (`LANGSMITH_API_KEY`) 或通过 `api_key` 参数直接传递。如果 `LangGraphClient` / `SyncLangGraphClient` 使用 `api_key` 参数初始化，也可以通过 `client` / `sync_client` 参数提供 API 密钥。

此外，您必须提供以下选项之一：

- `url`: 您要交互的部署的 URL。如果您传递 `url` 参数，则将使用提供的 URL、标头（如果提供）和默认配置值（例如，超时等）创建同步和异步客户端。
- `client`: 一个 `LangGraphClient` 实例，用于与部署进行异步交互（例如，使用 `.astream()`, `.ainvoke()`, `.aget_state()`, `.aupdate_state()` 等）。
- `sync_client`: 一个 `SyncLangGraphClient` 实例，用于与部署进行同步交互（例如，使用 `.stream()`, `.invoke()`, `.get_state()`, `.update_state()` 等）。

!!! Note

    如果您同时传递 `client` 或 `sync_client` 和 `url` 参数，它们将优先于 `url` 参数。如果未提供 `client` / `sync_client` / `url` 参数中的任何一个，`RemoteGraph` 将在运行时引发 `ValueError`。


### 使用 URL

=== "Python"

    ```python
    from langgraph.pregel.remote import RemoteGraph

    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    remote_graph = RemoteGraph(graph_name, url=url)
    ```

=== "JavaScript"

    ```ts
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const url = `<DEPLOYMENT_URL>`;
    const graphName = "agent";
    const remoteGraph = new RemoteGraph({ graphId: graphName, url });
    ```

### 使用客户端

=== "Python"

    ```python
    from langgraph_sdk import get_client, get_sync_client
    from langgraph.pregel.remote import RemoteGraph

    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    client = get_client(url=url)
    sync_client = get_sync_client(url=url)
    remote_graph = RemoteGraph(graph_name, client=client, sync_client=sync_client)
    ```

=== "JavaScript"

    ```ts
    import { Client } from "@langchain/langgraph-sdk";
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const client = new Client({ apiUrl: `<DEPLOYMENT_URL>` });
    const graphName = "agent";
    const remoteGraph = new RemoteGraph({ graphId: graphName, client });
    ```

## 调用图表

由于 `RemoteGraph` 是一个实现了与 `CompiledGraph` 相同方法的 `Runnable`，因此您可以像平常使用编译后的图表一样与它进行交互，即通过调用 `.invoke()`, `.stream()`, `.get_state()`, `.update_state()` 等（以及它们的异步对应项）。

### 异步调用

!!! Note

    要异步使用图表，您必须在初始化 `RemoteGraph` 时提供 `url` 或 `client`。

=== "Python"

    ```python
    # 调用图表
    result = await remote_graph.ainvoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    })

    # 流式输出图表
    async for chunk in remote_graph.astream({
        "messages": [{"role": "user", "content": "what's the weather in la"}]
    }):
        print(chunk)
    ```

=== "JavaScript"

    ```ts
    // 调用图表
    const result = await remoteGraph.invoke({
        messages: [{role: "user", content: "what's the weather in sf"}]
    })

    // 流式输出图表
    for await (const chunk of await remoteGraph.stream({
        messages: [{role: "user", content: "what's the weather in la"}]
    })):
        console.log(chunk)
    ```

### 同步调用

!!! Note

    要同步使用图表，您必须在初始化 `RemoteGraph` 时提供 `url` 或 `sync_client`。

=== "Python"

    ```python
    # 调用图表
    result = remote_graph.invoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    })

    # 流式输出图表
    for chunk in remote_graph.stream({
        "messages": [{"role": "user", "content": "what's the weather in la"}]
    }):
        print(chunk)
    ```

## 按线程级别持久化

默认情况下，图表运行（即 `.invoke()` 或 `.stream()` 调用）是无状态的 - 图表的检查点和最终状态不会被持久化。如果您想持久化图表运行的输出（例如，启用人工介入功能），您可以创建一个线程并通过 `config` 参数提供线程 ID，就像您对普通的编译图表一样：

=== "Python"

    ```python
    from langgraph_sdk import get_sync_client
    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    sync_client = get_sync_client(url=url)
    remote_graph = RemoteGraph(graph_name, url=url)

    # 创建一个线程（或使用现有线程）
    thread = sync_client.threads.create()

    # 使用线程配置调用图表
    config = {"configurable": {"thread_id": thread["thread_id"]}}
    result = remote_graph.invoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    }, config=config)

    # 验证状态是否已持久化到线程
    thread_state = remote_graph.get_state(config)
    print(thread_state)
    ```

=== "JavaScript"

    ```ts
    import { Client } from "@langchain/langgraph-sdk";
    import { RemoteGraph } from "@langchain/langgraph/remote";

    const url = `<DEPLOYMENT_URL>`;
    const graphName = "agent";
    const client = new Client({ apiUrl: url });
    const remoteGraph = new RemoteGraph({ graphId: graphName, url });

    // 创建一个线程（或使用现有线程）
    const thread = await client.threads.create();

    // 使用线程配置调用图表
    const config = { configurable: { thread_id: thread.thread_id }};
    const result = await remoteGraph.invoke({
      messages: [{ role: "user", content: "what's the weather in sf" }],
    }, config);

    // 验证状态是否已持久化到线程
    const threadState = await remoteGraph.getState(config);
    console.log(threadState);
    ```

## 作为子图使用

!!! Note

    如果需要将 `RemoteGraph` 用作具有 `checkpointer` 的图表的子图节点，请确保使用 UUID 作为线程 ID。


由于 `RemoteGraph` 的行为与普通 `CompiledGraph` 相同，它也可以用作另一个图表中的子图。例如：

=== "Python"

    ```python
    from langgraph_sdk import get_sync_client
    from langgraph.graph import StateGraph, MessagesState, START
    from typing import TypedDict

    url = <DEPLOYMENT_URL>
    graph_name = "agent"
    remote_graph = RemoteGraph(graph_name, url=url)

    # 定义父图
    builder = StateGraph(MessagesState)
    # 直接添加远程图作为节点
    builder.add_node("child", remote_graph)
    builder.add_edge(START, "child")
    graph = builder.compile()

    # 调用父图
    result = graph.invoke({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    })
    print(result)

    # 流式输出父图和子图
    for chunk in graph.stream({
        "messages": [{"role": "user", "content": "what's the weather in sf"}]
    }, subgraphs=True):
        print(chunk)
    ```

=== "JavaScript"

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
      .compile()

    // 调用父图
    const result = await graph.invoke({
      messages: [{ role: "user", content: "what's the weather in sf" }]
    });
    console.log(result);

    // 流式输出父图和子图
    for await (const chunk of await graph.stream({
      messages: [{ role: "user", content: "what's the weather in la" }]
    }, { subgraphs: true })) {
      console.log(chunk);
    }
    ```