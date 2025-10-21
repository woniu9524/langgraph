---
search:
  boost: 2
---

# LangGraph SDK

:::python
LangGraph 平台提供了一个 Python SDK，用于与 [LangGraph Server](./langgraph_server.md) 进行交互。

!!! tip "Python SDK 参考"

    有关 Python SDK 的详细信息，请参阅 [Python SDK 参考文档](../cloud/reference/sdk/python_sdk_ref.md)。

## 安装

您可以使用以下命令安装 LangGraph SDK：

```bash
pip install langgraph-sdk
```

## Python 同步与异步

Python SDK 提供了同步 (`get_sync_client`) 和异步 (`get_client`) 客户端，用于与 LangGraph Server 进行交互：

=== "Sync"

    ```python
    from langgraph_sdk import get_sync_client

    client = get_sync_client(url=..., api_key=...)
    client.assistants.search()
    ```

=== "Async"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=..., api_key=...)
    await client.assistants.search()
    ```

## 了解更多

- [Python SDK 参考](../cloud/reference/sdk/python_sdk_ref.md)
- [LangGraph CLI API Reference](../cloud/reference/cli.md)
  :::

:::js
LangGraph 平台提供了一个 JS/TS SDK，用于与 [LangGraph Server](./langgraph_server.md) 进行交互。

## 安装

您可以使用以下命令将 LangGraph SDK 添加到您的项目中：

```bash
npm install @langchain/langgraph-sdk
```

## 了解更多

- [LangGraph CLI API Reference](../cloud/reference/cli.md)
  :::