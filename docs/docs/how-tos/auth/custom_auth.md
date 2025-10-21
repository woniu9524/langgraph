# 添加自定义身份验证

!!! tip "先决条件"

    本指南假定您熟悉以下概念：

      *  [**身份验证与访问控制**](../../concepts/auth.md)
      *  [**LangGraph 平台**](../../concepts/langgraph_platform.md)

    如需更详尽的教程，请参阅[**设置自定义身份验证**](../../tutorials/auth/getting_started.md)教程。

???+ note "按部署类型支持"

    所有在 **托管 LangGraph 平台** 以及 **企业版** 自托管计划中的部署均支持自定义身份验证。

本指南将展示如何为您的 LangGraph 平台应用程序添加自定义身份验证。本指南适用于 LangGraph 平台和自托管部署。不适用于您自己在自定义服务器中独立使用 LangGraph 开源库的情况。

!!! note

    所有**托管 LangGraph 平台**部署以及**企业版**自托管计划均支持自定义身份验证。

## 为您的部署添加自定义身份验证

要利用自定义身份验证并在您的部署中访问用户级别元数据，请设置自定义身份验证，以通过自定义身份验证处理程序自动填充 `config["configurable"]["langgraph_auth_user"]` 对象。然后，您就可以在图中使用 `langgraph_auth_user` 键访问此对象，以[允许代理代表用户执行已授权操作](#enable-agent-authentication)。

:::python

1.  实现身份验证：

    !!! note

        如果没有自定义的 `@auth.authenticate` 处理程序，LangGraph 只能看到 API 密钥所有者（通常是开发者），因此请求不会限定到单个最终用户。要传递自定义令牌，您必须实现自己的处理程序。

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
            # ... 在此添加自定义字段/密钥
        }
    ```

    1. 此处理程序接收请求（标头等），验证用户，并返回一个至少包含身份标识符的字典。
    2. 您可以添加任何所需的自定义字段（例如，OAuth 令牌、角色、组织 ID 等）。

2.  在您的 `langgraph.json` 中，添加指向您的身份验证文件的路径：

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

3.  设置服务器中的身份验证后，请求必须根据您选择的方案包含所需的授权信息。假设您正在使用 JWT 令牌身份验证，您可以通过以下任何一种方法访问您的部署：

    === "Python 客户端"

        ```python
        from langgraph_sdk import get_client

        my_token = "your-token" # 实际上，您将使用您的身份验证提供程序生成签名令牌
        client = get_client(
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await client.threads.search()
        ```

    === "Python RemoteGraph"

        ```python
        from langgraph.pregel.remote import RemoteGraph

        my_token = "your-token" # 实际上，您将使用您的身份验证提供程序生成签名令牌
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```
        ```python
        from langgraph.pregel.remote import RemoteGraph

        my_token = "your-token" # 实际上，您将使用您的身份验证提供程序生成签名令牌
        remote_graph = RemoteGraph(
            "agent",
            url="http://localhost:2024",
            headers={"Authorization": f"Bearer {my_token}"}
        )
        threads = await remote_graph.ainvoke(...)
        ```

    === "CURL"

        ```bash
        curl -H "Authorization: Bearer ${your-token}" http://localhost:2024/threads
        ```

## 启用代理身份验证

在[身份验证](#add-custom-authentication-to-your-deployment)之后，平台会创建一个特殊的配置对象（`config`），该对象会传递给 LangGraph 平台部署。此对象包含当前用户信息，包括您从 `@auth.authenticate` 处理程序返回的任何自定义字段。

要允许代理代表用户执行已授权操作，请通过 `langgraph_auth_user` 键在您的图中访问此对象：

```python
def my_node(state, config):
    user_config = config["configurable"].get("langgraph_auth_user")
    # 令牌是在 @auth.authenticate 函数期间解析的
    token = user_config.get("github_token","")
    ...
```

!!! note

    从安全的密钥存储中获取用户凭据。不建议将密钥存储在图状态中。

### 授权 Studio 用户

默认情况下，如果您在资源上添加了自定义授权，这也将适用于从 Studio 进行的交互。如果需要，您可以通过检查 [is_studio_user()](../../reference/functions/sdk_auth.isStudioUser.html) 来区分已登录的 Studio 用户。

!!! note
    `is_studio_user` 已在 langgraph-sdk 的 0.1.73 版本中添加。如果您使用的是旧版本，仍然可以检查 `isinstance(ctx.user, StudioUser)`。

```python
from langgraph_sdk.auth import is_studio_user, Auth
auth = Auth()

# ... 设置 authenticate 等。

@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,
    value: dict  # 要发送到此访问方法的负载
) -> dict:  # 返回一个限制对资源访问的过滤器字典
    if is_studio_user(ctx.user):
        return {}

    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)
    return filters
```

仅当您希望允许开发者访问在托管 LangGraph 平台 SaaS 上部署的图时才使用此方法。

:::

:::js

1.  实现身份验证：

    !!! note

        如果没有自定义的 `authenticate` 处理程序，LangGraph 只能看到 API 密钥所有者（通常是开发者），因此请求不会限定到单个最终用户。要传递自定义令牌，您必须实现自己的处理程序。

    ```typescript
    import { Auth, HTTPException } from "@langchain/langgraph-sdk/auth";

    const auth = new Auth()
      .authenticate(async (request) => {
        const authorization = request.headers.get("Authorization");
        const token = authorization?.split(" ")[1]; // "Bearer <token>"
        if (!token) {
          throw new HTTPException(401, "未提供令牌");
        }
        try {
          const user = await verifyToken(token);
          return user;
        } catch (error) {
          throw new HTTPException(401, "无效令牌");
        }
      })
      // 添加授权规则以实际控制对资源的访问
      .on("*", async ({ user, value }) => {
        const filters = { owner: user.identity };
        const metadata = value.metadata ?? {};
        metadata.update(filters);
        return filters;
      })
      // 假设您在存储中按 (user_id, resource_type, resource_id) 的方式组织信息
      .on("store", async ({ user, value }) => {
        const namespace = value.namespace;
        if (namespace[0] !== user.identity) {
          throw new HTTPException(403, "未授权");
        }
      });
    ```

    1. 此处理程序接收请求（标头等），验证用户，并返回一个至少包含身份标识符的对象。
    2. 您可以添加任何所需的自定义字段（例如，OAuth 令牌、角色、组织 ID 等）。

2.  在您的 `langgraph.json` 中，添加指向您的身份验证文件的路径：

    ```json hl_lines="7-9"
    {
      "dependencies": ["."],
      "graphs": {
        "agent": "./agent.ts:graph"
      },
      "env": ".env",
      "auth": {
        "path": "./auth.ts:my_auth"
      }
    }
    ```

3.  设置服务器中的身份验证后，请求必须根据您选择的方案包含所需的授权信息。假设您正在使用 JWT 令牌身份验证，您可以通过以下任何一种方法访问您的部署：

    === "SDK 客户端"

        ```javascript
        import { Client } from "@langchain/langgraph-sdk";

        const my_token = "your-token"; // 实际上，您将使用您的身份验证提供程序生成签名令牌
        const client = new Client({
          apiUrl: "http://localhost:2024",
          defaultHeaders: { Authorization: `Bearer ${my_token}` },
        });
        const threads = await client.threads.search();
        ```

    === "RemoteGraph"

        ```javascript
        import { RemoteGraph } from "@langchain/langgraph/remote";

        const my_token = "your-token"; // 实际上，您将使用您的身份验证提供程序生成签名令牌
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

在[身份验证](#add-custom-authentication-to-your-deployment)之后，平台会创建一个特殊的配置对象（`config`），该对象会传递给 LangGraph 平台部署。此对象包含当前用户信息，包括您从 `authenticate` 处理程序返回的任何自定义字段。

要允许代理代表用户执行已授权操作，请通过 `langgraph_auth_user` 键在您的图中访问此对象：

```ts
async function myNode(state, config) {
  const userConfig = config["configurable"]["langgraph_auth_user"];
  // 令牌是在 authenticate 函数期间解析的
  const token = userConfig["github_token"];
  ...
}
```

!!! note

    从安全的密钥存储中获取用户凭据。不建议将密钥存储在图状态中。

:::

## 了解更多

- [身份验证与访问控制](../../concepts/auth.md)
- [LangGraph 平台](../../concepts/langgraph_platform.md)
- [设置自定义身份验证教程](../../tutorials/auth/getting_started.md)