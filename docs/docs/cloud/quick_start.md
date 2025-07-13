# 部署快速入门

本指南将向您展示如何为云部署设置和使用 LangGraph Platform。

## 先决条件

在开始之前，请确保您已满足以下条件：

- 一个 [GitHub 账户](https://github.com/)
- 一个 [LangSmith 账户](https://smith.langchain.com/) – 免费注册

## 1. 在 GitHub 上创建仓库

要将应用程序部署到 **LangGraph Platform**，您的应用程序代码必须位于 GitHub 仓库中。支持公共仓库和私有仓库。对于本快速入门，请使用 [`new-langgraph-project` 模板](https://github.com/langchain-ai/react-agent) 作为您的应用程序：

1. 转到 [`new-langgraph-project` 仓库](https://github.com/langchain-ai/new-langgraph-project) 或 [`new-langgraphjs-project` 模板](https://github.com/langchain-ai/new-langgraphjs-project)。
1. 点击右上角的 `Fork` 按钮，将仓库 fork 到您的 GitHub 账户。
1. 点击 **Create fork**。

## 2. 部署到 LangGraph Platform

1. 登录 [LangSmith](https://smith.langchain.com/)。
1. 在左侧边栏中，选择 **Deployments**。
1. 点击 **+ New Deployment** 按钮。将打开一个窗格，您可以在其中填写必填字段。
1. 如果您是首次使用或正在添加 ранее 未连接过的私有仓库，请点击 **Import from GitHub** 按钮并按照说明连接您的 GitHub 账户。
1. 选择您的 New LangGraph Project 仓库。
1. 点击 **Submit** 进行部署。

    这可能需要大约 15 分钟才能完成。您可以在 **Deployment details** 视图中查看状态。

> **注意：** 此处可能存在翻译不准确的地方，因为我无法访问您的项目。

## 3. 在 LangGraph Studio 中测试您的应用程序

部署应用程序后：

1. 选择您刚创建的部署以查看更多详细信息。
1. 点击右上角的 **LangGraph Studio** 按钮。

    LangGraph Studio 将打开以显示您的图形。

    <figure markdown="1">
    [![image](deployment/img/langgraph_studio.png){: style="max-height:400px"}](deployment/img/langgraph_studio.png)
    <figcaption>
        在 LangGraph Studio 中运行的示例图形。
    </figcaption>
    </figure>

## 4. 获取部署的 API URL

1. 在 LangGraph 的 **Deployment details** 视图中，点击 **API URL** 将其复制到剪贴板。
1. 点击 `URL` 将其复制到剪贴板。

## 5. 测试 API

现在您可以测试 API：

=== "Python SDK (Async)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    1. 向助手发送消息（无线程运行）：

        ```python
        from langgraph_sdk import get_client

        client = get_client(url="your-deployment-url", api_key="your-langsmith-api-key")

        async for chunk in client.runs.stream(
            None,  # Threadless run
            "agent", # Assistant name. Defined in langgraph.json.
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="updates",
        ):
            print(f"Receiving new event of type: {chunk.event}...")
            print(chunk.data)
            print("\n\n")
        ```

=== "Python SDK (Sync)"

    1. 安装 LangGraph Python SDK：

        ```shell
        pip install langgraph-sdk
        ```

    1. 向助手发送消息（无线程运行）：

        ```python
        from langgraph_sdk import get_sync_client

        client = get_sync_client(url="your-deployment-url", api_key="your-langsmith-api-key")

        for chunk in client.runs.stream(
            None,  # Threadless run
            "agent", # Assistant name. Defined in langgraph.json.
            input={
                "messages": [{
                    "role": "human",
                    "content": "What is LangGraph?",
                }],
            },
            stream_mode="updates",
        ):
            print(f"Receiving new event of type: {chunk.event}...")
            print(chunk.data)
            print("\n\n")
        ```

=== "JavaScript SDK"

    1. 安装 LangGraph JS SDK

        ```shell
        npm install @langchain/langgraph-sdk
        ```

    1. 向助手发送消息（无线程运行）：

        ```js
        const { Client } = await import("@langchain/langgraph-sdk");

        const client = new Client({ apiUrl: "your-deployment-url", apiKey: "your-langsmith-api-key" });

        const streamResponse = client.runs.stream(
            null, // Threadless run
            "agent", // Assistant ID
            {
                input: {
                    "messages": [
                        { "role": "user", "content": "What is LangGraph?"}
                    ]
                },
                streamMode: "messages",
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
        --url <DEPLOYMENT_URL>/runs/stream \
        --header 'Content-Type: application/json' \
        --header "X-Api-Key: <LANGSMITH API KEY> \
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
            \"stream_mode\": \"updates\"
        }"
    ```

## 后续步骤

恭喜！您已使用 LangGraph Platform 部署了应用程序。

您可以查看以下其他资源：

- [LangGraph Platform 概览](../concepts/langgraph_platform.md)
- [部署选项](../concepts/deployment_options.md)