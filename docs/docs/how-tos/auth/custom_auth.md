# 添加自定义身份验证

本指南将介绍如何为 LangGraph Platform 应用程序添加自定义身份验证。本指南适用于 LangGraph Platform 和自托管部署。它不适用于您自己自定义服务器中 LangGraph 开源库的独立使用。

!!! note

    自定义身份验证支持所有**托管的 LangGraph Platform** 部署以及**企业版**的自托管计划。它不支持**精简版**的自托管计划。

## 为您的部署添加自定义身份验证

要利用自定义身份验证并在您的部署中访问用户级别的元数据，请设置自定义身份验证，通过自定义身份验证处理程序自动填充 `config["configurable"]["langgraph_auth_user"]` 对象。然后，您可以使用 `langgraph_auth_user` 键在图中访问此对象，以[允许代理代表用户执行身份验证操作](#enable-agent-authentication)。

1. 实现身份验证：

    !!! note

        如果没有自定义的 `@auth.authenticate` 处理程序，LangGraph 只会看到 API 密钥的所有者（通常是开发者），因此请求不会按个体最终用户进行范围界定。要传播自定义令牌，您必须实现自己的处理程序。

    ```python
    from langgraph_sdk import Auth
    import requests

    auth = Auth()

    def is_valid_key(api_key: str) -> bool:
        is_valid = # 您的 API 密钥验证逻辑
        return is_valid

    @auth.authenticate # (1)!
    async def authenticate(headers: dict) -> Auth.types.MinimalUserDict:
        api_key = headers.get("x-api-key")
        if not api_key or not is_valid_key(api_key):
            raise Auth.exceptions.HTTPException(status_code=401, detail="无效的 API 密钥")
        
        # 从您的密钥存储中获取用户特定令牌  
        user_tokens = await fetch_user_tokens(api_key)

        return { # (2)!
            "identity": api_key,  # 从 LangSmith 获取用户 ID 
            "github_token" : user_tokens.github_token
            "jira_token" : user_tokens.jira_token
            # ... 自定义字段/密钥在这里
        }
    ```

    1. 此处理程序接收请求（标题、等），验证用户，并返回一个至少包含身份字段的字典。
    2. 您可以根据需要添加任何自定义字段（例如，OAuth 令牌、角色、组织 ID 等）。

2. 在您的 `langgraph.json` 中，添加您的身份验证文件的路径：

    ```json hl_lines="7-9"
    {
    "dependencies": ["."],
    "graphs": {
        "agent": "./agent.py:graph"
    },
    "env": ".env",
    "auth": {
        "path": "./auth.py:my_auth"
    }
    }
    ```

3. 在服务器中设置身份验证后，请求必须根据您选择的方案包含所需的授权信息。假设您正在使用 JWT 令牌身份验证，您可以通过以下任何方法访问您的部署：

    === "Python 客户端"

        ```python
        from langgraph_sdk import get_client

        my_token = "your-token" # 实际中，您将生成一个已签名的令牌，包含您的身份验证提供商
        client = get_client(
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await client.threads.search()
        ```

    === "Python RemoteGraph"

        ```python
        from langgraph.pregel.remote import RemoteGraph
        
        my_token = "your-token" # 实际中，您将生成一个已签名的令牌，包含您的身份验证提供商
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```

    === "JavaScript 客户端"

        ```javascript
        import { Client } from "@langchain/langgraph-sdk";

        const my_token = "your-token"; // 实际中，您将生成一个已签名的令牌，包含您的身份验证提供商
        const client = new Client({
        apiUrl: "http://localhost:2024",
        defaultHeaders: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await client.threads.search();
        ```

    === "JavaScript RemoteGraph"

        ```javascript
        import { RemoteGraph } from "@langchain/langgraph/remote";

        const my_token = "your-token"; // 实际中，您将生成一个已签名的令牌，包含您的身份验证提供商
        const remoteGraph = new RemoteGraph({
        graphId: "agent",
        url: "http://localhost:2024",
        headers: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await remoteGraph.invoke(...);
        ```

    === "CURL"

        ```bash
        curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
        ```

## 启用代理身份验证

在进行[身份验证](#add-custom-authentication-to-your-deployment)后，平台会创建一个特殊的配置对象 (`config`) 并将其传递给 LangGraph Platform 部署。此对象包含有关当前用户的信息，包括您从 `@auth.authenticate` 处理程序返回的任何自定义字段。

要允许代理代表用户执行身份验证操作，请在图中访问此对象，使用 `langgraph_auth_user` 键：

```python
def my_node(state, config):
    user_config = config["configurable"].get("langgraph_auth_user")
    # token 在 @auth.authenticate 函数期间已解析
    token = user_config.get("github_token","") 
    ...
```

!!! note
    从安全的密钥存储中获取用户凭据。不建议将密钥存储在图状态中。

## 了解更多

* [身份验证与访问控制](../../concepts/auth.md)
* [LangGraph Platform](../../concepts/langgraph_platform.md)
* [设置自定义身份验证教程](../../tutorials/auth/getting_started.md)