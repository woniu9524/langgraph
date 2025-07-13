---
search:
  boost: 2
---

# LangGraph SDK

LangGraph Platform 提供了与 [LangGraph Server](./langgraph_server.md) 交互的 Python SDK。

!!! tip "Python SDK 参考"
  
    有关 Python SDK 的详细信息，请参阅 [Python SDK 参考文档](../cloud/reference/sdk/python_sdk_ref.md)。

## 安装

您可以使用适用于您语言的相应包管理器来安装这些包：

=== "Python"
    ```bash
    pip install langgraph-sdk
    ```

=== "JS"
    ```bash
    yarn add @langchain/langgraph-sdk
    ```

## Python 同步 vs. 异步

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
- [LangGraph CLI API 参考](../cloud/reference/cli.md)
- [JS/TS SDK 参考](../cloud/reference/sdk/js_ts_sdk_ref.md)