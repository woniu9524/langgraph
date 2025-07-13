# MCP

[模型上下文协议 (MCP)](https://modelcontextprotocol.io/introduction) 是一个开放协议，用于标准化应用程序如何向语言模型提供工具和上下文。LangGraph 代理可以通过 `langchain-mcp-adapters` 库使用 MCP 服务器上定义的工具。

![MCP](../agents/assets/mcp.png)

安装 `langchain-mcp-adapters` 库即可在 LangGraph 中使用 MCP 工具：

```bash
pip install langchain-mcp-adapters
```

## 认证到 MCP 服务器

您可以设置 [自定义认证中间件](../how-tos/auth/custom_auth.md)，以通过 MCP 服务器对用户进行身份验证，从而在您的 LangGraph Platform 部署中访问用户范围的工具。

!!! note
    自定义认证是 LangGraph Platform 的一项功能。

此流程的示例架构：

```mermaid
sequenceDiagram
  %% Actors
  participant ClientApp as 客户端
  participant AuthProv  as 认证提供程序
  participant LangGraph as LangGraph 后端
  participant SecretStore as 密码存储
  participant MCPServer as MCP 服务器

  %% Platform login / AuthN
  ClientApp  ->> AuthProv: 1. 登录 (用户名/密码)
  AuthProv   -->> ClientApp: 2. 返回 token
  ClientApp  ->> LangGraph: 3. 使用 token 请求

  Note over LangGraph: 4. 验证 token (@auth.authenticate)
  LangGraph  -->> AuthProv: 5. 获取用户信息
  AuthProv   -->> LangGraph: 6. 确认有效性

  %% Fetch user tokens from secret store
  LangGraph  ->> SecretStore: 6a. 获取用户 token
  SecretStore -->> LangGraph: 6b. 返回 token

  Note over LangGraph: 7. 应用访问控制 (@auth.on.*)

  %% MCP round-trip
  Note over LangGraph: 8. 使用用户 token 构建 MCP 客户端
  LangGraph  ->> MCPServer: 9. 调用 MCP 工具 (带 header)
  Note over MCPServer: 10. MCP 验证 header 并运行工具
  MCPServer  -->> LangGraph: 11. 工具响应

  %% Return to caller
  LangGraph  -->> ClientApp: 12. 返回资源 / 工具输出
```

有关更多信息，请参阅 [LangGraph Server 中的 MCP 端点](../concepts/server-mcp.md#use-user-scoped-mcp-tools-in-your-deployment)。