# 文档 API 身份验证（OpenAPI）

本指南展示了如何为 LangGraph 平台 API 文档自定义 OpenAPI 安全模式。完善的安全模式文档有助于 API 消费者了解如何通过 API 进行身份验证，甚至可以自动生成客户端。有关 LangGraph 身份验证系统的更多详细信息，请参阅[身份验证与访问控制概念指南](../../concepts/auth.md)。

!!! note "实现 vs 文档"
    本指南仅涵盖如何在 OpenAPI 中记录你的安全要求。要实现实际的身份验证逻辑，请参阅[如何添加自定义身份验证](./custom_auth.md)。

本指南适用于所有 LangGraph 平台部署（云部署和自托管）。如果你未使用 LangGraph 平台，那么它不适用于 LangGraph 开源库的使用。

## 默认模式

默认安全模式因部署类型而异：

=== "LangGraph 平台"

默认情况下，LangGraph 平台要求在 `x-api-key` 请求头中提供 LangSmith API 密钥：

```yaml
components:
  securitySchemes:
    apiKeyAuth:
      type: apiKey
      in: header
      name: x-api-key
security:
  - apiKeyAuth: []
```

当使用 LangGraph SDK 时，这可以从环境变量中推断出来。

=== "自托管"

默认情况下，自托管部署没有安全模式。这意味着它们只能部署在安全网络上或通过身份验证进行部署。要添加自定义身份验证，请参阅[如何添加自定义身份验证](./custom_auth.md)。

## 自定义安全模式

要自定义 OpenAPI 文档中的安全模式，请在 `langgraph.json` 的 `auth` 配置中添加 `openapi` 字段。请记住，这只会更新 API 文档 — 你还必须按照[如何添加自定义身份验证](./custom_auth.md)中的说明实现相应的身份验证逻辑。

请注意，LangGraph 平台不提供身份验证端点 — 你需要在客户端应用程序中处理用户身份验证，并将生成的凭证传递给 LangGraph API。

=== "OAuth2 令牌持有人"

    ```json
    {
      "auth": {
        "path": "./auth.py:my_auth",  // 在此处实现身份验证逻辑
        "openapi": {
          "securitySchemes": {
            " OAuth2": {
              "type": "oauth2",
              "flows": {
                "implicit": {
                  "authorizationUrl": "https://your-auth-server.com/oauth/authorize",
                  "scopes": {
                    "me": "读取当前用户信息",
                    "threads": "访问创建和管理线程"
                  }
                }
              }
            }
          },
          "security": [
            {"OAuth2": ["me", "threads"]}
          ]
        }
      }
    }
    ```

=== "API 密钥"

    ```json
    {
      "auth": {
        "path": "./auth.py:my_auth",  // 在此处实现身份验证逻辑
        "openapi": {
          "securitySchemes": {
            "apiKeyAuth": {
              "type": "apiKey",
              "in": "header",
              "name": "X-API-Key"
            }
          },
          "security": [
            {"apiKeyAuth": []}
          ]
        }
      }
    }
    ```

## 测试

更新配置后：

1. 部署你的应用程序
2. 访问 `/docs` 查看更新后的 OpenAPI 文档
3. 使用来自身份验证服务器的凭证尝试使用端点（请确保你已首先实现了身份验证逻辑）