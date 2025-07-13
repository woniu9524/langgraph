# 运行本地服务器

本指南将向您展示如何运行 LangGraph 应用程序。

## 前提条件

开始之前，请确保您具备以下条件：

- [LangSmith](https://smith.langchain.com/settings) 的 API 密钥 - 可免费注册

## 1. 安装 LangGraph CLI

=== "Python 服务器"

    ```shell
    # 需要 Python >= 3.11。

    pip install --upgrade "langgraph-cli[inmem]"
    ```

=== "Node 服务器"

    ```shell
    npx @langchain/langgraph-cli
    ```

## 2. 创建 LangGraph 应用 🌱

从 [`new-langgraph-project-python` 模板](https://github.com/langchain-ai/new-langgraph-project) 或 [`new-langgraph-project-js` 模板](https://github.com/langchain-ai/new-langgraphjs-project) 创建一个新应用。此模板演示了一个单节点应用程序，您可以根据自身逻辑进行扩展。

=== "Python 服务器"

    ```shell
    langgraph new path/to/your/app --template new-langgraph-project-python
    ```

=== "Node 服务器"

    ```shell
    langgraph new path/to/your/app --template new-langgraph-project-js
    ```

!!! tip "其他模板"

    如果您在使用 `langgraph new` 时不指定模板，系统将出现一个交互式菜单，允许您从可用模板列表中进行选择。

## 3. 安装依赖项

在新的 LangGraph 应用的根目录下，以 `edit` 模式安装依赖项，以便服务器使用您的本地更改：

=== "Python 服务器"

    ```shell
    cd path/to/your/app
    pip install -e .
    ```

=== "Node 服务器"

    ```shell
    cd path/to/your/app
    yarn install
    ```

## 4. 创建 `.env` 文件

您会在新的 LangGraph 应用的根目录下找到一个 `.env.example` 文件。请在您的新 LangGraph 应用的根目录下创建一个 `.env` 文件，并将 `.env.example` 文件中的内容复制进去，填入必要的 API 密钥：

```bash
LANGSMITH_API_KEY=lsv2...
```

## 5. 启动 LangGraph 服务器 🚀

在本地启动 LangGraph API 服务器：

=== "Python 服务器"

    ```shell
    langgraph dev
    ```

=== "Node 服务器"

    ```shell
    npx @langchain/langgraph-cli dev
    ```

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

`langgraph dev` 命令以内存模式启动 LangGraph 服务器。此模式适用于开发和测试目的。生产使用时，请使用具有持久存储后端访问权限的 LangGraph 服务器进行部署。有关更多信息，请参阅 [部署选项](../../concepts/deployment_options.md)。

## 6. 在 LangGraph Studio 中测试您的应用

[LangGraph Studio](../../concepts/langgraph_studio.md) 是一个专门的 UI，您可以将其连接到 LangGraph API 服务器，以便在本地可视化、交互和调试您的应用程序。通过访问 `langgraph dev` 命令输出中提供的 URL，在 LangGraph Studio 中测试您的图：

```
>    - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
```

对于运行在自定义主机/端口上的 LangGraph 服务器，请更新 `baseURL` 参数。

??? info "Safari 兼容性"

    使用 `--tunnel` 标志运行您的命令以创建安全隧道，因为 Safari 在连接到 localhost 服务器时存在限制：

    ```shell
    langgraph dev --tunnel
    ```

## 7. 测试 API

=== "Python SDK (异步)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    1. 向助手发送消息（无状态运行）：

        ```python
        from langgraph_sdk import get_client
        import asyncio

        client = get_client(url="http://localhost:2024")

        async def main():
            async for chunk in client.runs.stream(
                None,  # 无状态运行
                "agent", # 助手的名称。在 langgraph.json 中定义。
                input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                    }],
                },
            ):
                print(f"接收到类型为 {chunk.event} 的新事件...")
                print(chunk.data)
                print("\n\n")

        asyncio.run(main())
        ```

=== "Python SDK (同步)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    1. 向助手发送消息（无状态运行）：

        ```python
        from langgraph_sdk import get_sync_client

        client = get_sync_client(url="http://localhost:2024")

        for chunk in client.runs.stream(
            None,  # 无状态运行
            "agent", # 助手的名称。在 langgraph.json 中定义。
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="messages-tuple",
        ):
            print(f"接收到类型为 {chunk.event} 的新事件...")
            print(chunk.data)
            print("\n\n")
        ```


=== "Javascript SDK"

    1. 安装 LangGraph JS SDK：

        ```shell
        npm install @langchain/langgraph-sdk
        ```

    1. 向助手发送消息（无状态运行）：

        ```js
        const { Client } = await import("@langchain/langgraph-sdk");

        // 仅当您在调用 langgraph dev 时更改了默认端口时才设置 apiUrl
        const client = new Client({ apiUrl: "http://localhost:2024"});

        const streamResponse = client.runs.stream(
            null, // 无状态运行
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

## 后续步骤

现在您已经在本地运行了 LangGraph 应用，请通过探索部署和高级功能来进一步深入学习：

- [部署快速入门](../../cloud/quick_start.md)：使用 LangGraph Platform 部署您的 LangGraph 应用。
- [LangGraph Platform 概述](../../concepts/langgraph_platform.md)：了解 LangGraph Platform 的基础概念。
- [LangGraph Server API 参考](../../cloud/reference/api/api_ref.html)：探索 LangGraph Server API 文档。
- [Python SDK 参考](../../cloud/reference/sdk/python_sdk_ref.md)：探索 Python SDK API 参考。
- [JS/TS SDK 参考](../../cloud/reference/sdk/js_ts_sdk_ref.md)：探索 JS/TS SDK API 参考。