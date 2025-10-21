# 运行本地服务器

本指南将指导你如何在本地运行 LangGraph 应用程序。

## 前提条件

在你开始之前，请确保你已具备以下条件：

- [LangSmith](https://smith.langchain.com/settings) 的 API 密钥（可免费注册）

## 1. 安装 LangGraph CLI

:::python

```shell
# 需要 Python >= 3.11。

pip install --upgrade "langgraph-cli[inmem]"
```

:::

:::js

```shell
npx @langchain/langgraph-cli
```

:::

## 2. 创建 LangGraph 应用 🌱

:::python
从 [`new-langgraph-project-python` 模板](https://github.com/langchain-ai/new-langgraph-project) 创建一个新应用。该模板演示了一个你可以扩展自定义逻辑的单节点应用程序。

```shell
langgraph new path/to/your/app --template new-langgraph-project-python
```

!!! tip "更多模板"

    如果你使用 `langgraph new` 时不指定模板，将会出现一个交互式菜单，让你选择可用模板列表中的一个。

:::

:::js
从 [`new-langgraph-project-js` 模板](https://github.com/langchain-ai/new-langgraph-project-js)。该模板演示了一个你可以扩展自定义逻辑的单节点应用程序。

```shell
npm create langgraph
```

:::

## 3. 安装依赖项

在新的 LangGraph 应用的根目录下，以“编辑”模式安装依赖项，以便服务器使用你的本地更改：

:::python

```shell
cd path/to/your/app
pip install -e .
```

:::

:::js

```shell
cd path/to/your/app
npm install
```

:::

## 4. 创建 `.env` 文件

你将在新的 LangGraph 应用的根目录下找到一个 `.env.example` 文件。在你的新 LangGraph 应用的根目录下创建一个 `.env` 文件，并将 `.env.example` 文件中的内容复制到其中，填入必要的 API 密钥：

```bash
LANGSMITH_API_KEY=lsv2...
```

## 5. 启动 LangGraph 服务器 🚀

在本地启动 LangGraph API 服务器：

:::python

```shell
langgraph dev
```

:::

:::js

```shell
npx @langchain/langgraph-cli dev
```

:::

示例输出：

```
>    Ready!
>
>    - API: [http://localhost:2024](http://localhost:2024/)
>
>    - Docs: http://localhost:2024/docs
>
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

`langgraph dev` 命令以内存模式启动 LangGraph 服务器。此模式适用于开发和测试目的。对于生产环境，请使用带有持久化存储后端的 LangGraph 服务器进行部署。有关更多信息，请参阅[部署选项](../../concepts/deployment_options.md)。

## 6. 在 LangGraph Studio 中测试你的应用程序

[LangGraph Studio](../../concepts/langgraph_studio.md) 是一个专门的 UI，你可以将其连接到 LangGraph API 服务器，以便在本地可视化、交互和调试你的应用程序。通过访问 `langgraph dev` 命令输出中提供的 URL 来在 LangGraph Studio 中测试你的图：

```
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

如果 LangGraph 服务器运行在自定义主机/端口上，请更新 `baseUrl` 参数。

??? info "Safari 兼容性"

    在命令中使用 `--tunnel` 标志来创建安全隧道，因为 Safari 在连接到本地服务器时存在一些限制：

    ```shell
    langgraph dev --tunnel
    ```

## 7. 测试 API

:::python
=== "Python SDK (async)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    1. 向助手发送消息（无会话运行）：

        ```python
        from langgraph_sdk import get_client
        import asyncio

        client = get_client(url="http://localhost:2024")

        async def main():
            async for chunk in client.runs.stream(
                None,  # 无会话运行
                "agent", # 助手的名称。在 langgraph.json 中定义。
                input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                    }],
                },
            ):
                print(f"Receiving new event of type: {chunk.event}...")
                print(chunk.data)
                print("\n\n")

        asyncio.run(main())
        ```

=== "Python SDK (sync)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    1. 向助手发送消息（无会话运行）：

        ```python
        from langgraph_sdk import get_sync_client

        client = get_sync_client(url="http://localhost:2024")

        for chunk in client.runs.stream(
            None,  # 无会话运行
            "agent", # 助手的名称。在 langgraph.json 中定义。
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="messages-tuple",
        ):
            print(f"Receiving new event of type: {chunk.event}...")
            print(chunk.data)
            print("\n\n")
        ```

=== "Rest API"

    ```bash
    curl -s --request POST \
        --url "http://localhost:2024/runs/stream" \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"messages-tuple\"
        }"
    ```

:::

:::js
=== "Javascript SDK"

    1. 安装 LangGraph JS SDK：

        ```shell
        npm install @langchain/langgraph-sdk
        ```

    1. 向助手发送消息（无会话运行）：

        ```js
        const { Client } = await import("@langchain/langgraph-sdk");

        // 仅当更改了调用 langgraph dev 时的默认端口时才设置 apiUrl
        const client = new Client({ apiUrl: "http://localhost:2024"});

        const streamResponse = client.runs.stream(
            null, // 无会话运行
            "agent", // 助手 ID
            {
                input: {
                    "messages": [
                        { "role": "user", "content": "What is LangGraph?"}
                    ]
                },
                streamMode: "messages-tuple",
            }
        );

        for await (const chunk of streamResponse) {
            console.log(`Receiving new event of type: ${chunk.event}...`);
            console.log(JSON.stringify(chunk.data));
            console.log("\n\n");
        }
        ```

=== "Rest API"

    ```bash
    curl -s --request POST \
        --url "http://localhost:2024/runs/stream" \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {
                \"messages\": [
                    {
                        \"role\": \"human\",
                        \"content\": \"What is LangGraph?\"
                    }
                ]
            },
            \"stream_mode\": \"messages-tuple\"
        }"
    ```

:::

## 下一步

现在你已在本地运行了 LangGraph 应用，可以通过探索部署和高级功能来进一步深入：

- [部署快速入门](../../cloud/quick_start.md)：使用 LangGraph Platform 部署你的 LangGraph 应用。
- [LangGraph Platform 概述](../../concepts/langgraph_platform.md)：了解 LangGraph Platform 的基础概念。
- [LangGraph 服务器 API 参考](../../cloud/reference/api/api_ref.html)：浏览 LangGraph 服务器 API 文档。

:::python

- [Python SDK 参考](../../cloud/reference/sdk/python_sdk_ref.md)：浏览 Python SDK API 参考。
  :::

:::js

- [JS/TS SDK 参考](../../cloud/reference/sdk/js_ts_sdk_ref.md)：浏览 JS/TS SDK API 参考。
  :::