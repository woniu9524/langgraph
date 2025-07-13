---
tags:
  - mcp
  - platform
hide:
  - tags
---

# LangGraph Server 中的 MCP 端点

[模型上下文协议 (MCP)](./mcp.md) 是一种用于以模型无关格式描述工具和数据源的开放协议，它使 LLM 能够通过结构化 API 来发现和使用它们。

[LangGraph Server](./langgraph_server.md) 使用 [Streamable HTTP 传输](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/transports/#streamable-http) 来实现 MCP。这使得 LangGraph **代理 (agents)** 可以作为 **MCP 工具**公开，从而可供支持 Streamable HTTP 的任何符合 MCP 的客户端使用。

MCP 端点位于 [LangGraph Server](./langgraph_server.md) 的 `/mcp` 路径下。

## 要求

要使用 MCP，请确保已安装以下依赖项：

- `langgraph-api >= 0.2.3`
- `langgraph-sdk >= 0.1.61`

使用以下命令安装它们：

```bash
pip install "langgraph-api>=0.2.3" "langgraph-sdk>=0.1.61"
```

## 用法概览

要启用 MCP：

- 升级到使用 langgraph-api>=0.2.3。如果您正在部署 LangGraph Platform，在创建新修订版时会自动完成此操作。
- MCP 工具（代理）将自动公开。
- 连接到任何支持 Streamable HTTP 的符合 MCP 的客户端。


### 客户端

使用符合 MCP 的客户端连接到 LangGraph 服务器。以下示例展示了如何使用不同的编程语言进行连接。

=== "JavaScript/TypeScript"

    ```bash
    npm install @modelcontextprotocol/sdk
    ```

    > **注意**
    > 将 `serverUrl` 替换为您的 LangGraph 服务器 URL，并根据需要配置身份验证标头。

    ```js
    import { Client } from "@modelcontextprotocol/sdk/client/index.js";
    import { StreamableHTTPClientTransport } from "@modelcontextprotocol/sdk/client/streamableHttp.js";

    // 连接到 LangGraph MCP 端点
    async function connectClient(url) {
        const baseUrl = new URL(url);
        const client = new Client({
            name: 'streamable-http-client',
            version: '1.0.0'
        });

        const transport = new StreamableHTTPClientTransport(baseUrl);
        await client.connect(transport);

        console.log("Connected using Streamable HTTP transport");
        console.log(JSON.stringify(await client.listTools(), null, 2));
        return client;
    }

    const serverUrl = "http://localhost:2024/mcp";

    connectClient(serverUrl)
        .then(() => {
            console.log("Client connected successfully");
        })
        .catch(error => {
            console.error("Failed to connect client:", error);
        });
    ```

=== "Python"


    使用以下命令安装适配器：

    ```bash
    pip install langchain-mcp-adapters
    ```

    以下是如何连接到远程 MCP 端点并使用代理作为工具的示例：

    ```python
    # 为 stdio 连接创建服务器参数
    from mcp import ClientSession
    from mcp.client.streamable_http import streamablehttp_client
    import asyncio

    from langchain_mcp_adapters.tools import load_mcp_tools
    from langgraph.prebuilt import create_react_agent

    server_params = {
        "url": "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
        "headers": {
            "X-Api-Key":"lsv2_pt_your_api_key"
        }
    }

    async def main():
        async with streamablehttp_client(**server_params) as (read, write, _):
            async with ClientSession(read, write) as session:
                # 初始化连接
                await session.initialize()

                # 加载远程图，就像它是一个工具一样
                tools = await load_mcp_tools(session)

                # 使用工具创建并运行一个 react 代理
                agent = create_react_agent("openai:gpt-4.1", tools)

                # 使用消息调用代理
                agent_response = await agent.ainvoke({"messages": "What can the finance agent do for me?"})
                print(agent_response)

    if __name__ == "__main__":
        asyncio.run(main())
    ```

## 将代理公開为 MCP 工具

部署后，您的代理将作为工具出现在 MCP 端点中，配置如下：

- **工具名称 (Tool name)**：代理的名称。
- **工具描述 (Tool description)**：代理的描述。
- **工具输入模式 (Tool input schema)**：代理的输入模式。

### 设置名称和描述

您可以在 `langgraph.json` 中设置代理的名称和描述：

```json
{
    "graphs": {
        "my_agent": {
            "path": "./my_agent/agent.py:graph",
            "description": "A description of what the agent does"
        }
    },
    "env": ".env"
}
```

部署后，您可以使用 LangGraph SDK 更新名称和描述。

### 模式 (Schema)

定义清晰、最小化的输入和输出模式，以避免向 LLM 公开不必要的内部复杂性。

默认的 [MessagesState](./low_level.md#messagesstate) 使用 `AnyMessage`，它支持许多消息类型，但对于直接暴露给 LLM 来说过于通用。

相反，请定义**自定义代理或工作流**，它们使用显式类型化的输入和输出结构。

例如，一个用于回答文档问题的示例工作流可能如下所示：

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 定义输入模式
class InputState(TypedDict):
    question: str

# 定义输出模式
class OutputState(TypedDict):
    answer: str

# 合并输入和输出
class OverallState(InputState, OutputState):
    pass

# 定义处理节点
def answer_node(state: InputState):
    # 替换为实际逻辑并执行有用的操作
    return {"answer": "bye", "question": state["question"]}

# 使用显式模式构建图
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)
builder.add_edge(START, "answer_node")
builder.add_edge("answer_node", END)
graph = builder.compile()

# 运行图
print(graph.invoke({"question": "hi"}))
```

有关更多详细信息，请参阅 [低级概念指南](https://langchain-ai.github.io/langgraph/concepts/low_level/#state)。

## 在您的部署中使用用户范围的 MCP 工具

!!! tip "先决条件"

    您已添加自己的 [自定义身份验证中间件](https://langchain-ai.github.io/langgraph/how-tos/auth/custom_auth/)，该中间件会填充 `langgraph_auth_user` 对象，使其可通过可配置上下文供图中的每个节点访问。

要使您的 LangGraph Platform 部署可用用户范围的工具，请从实现以下代码段开始：

```python
from langchain_mcp_adapters.client import MultiServerMCPClient

def mcp_tools_node(state, config):
    user = config["configurable"].get("langgraph_auth_user")
		 # 例如，user["github_token"], user["email"] 等
		
    client = MultiServerMCPClient({
        "github": {
            "transport": "streamable_http", # (1)
            "url": "https://my-github-mcp-server/mcp", # (2)
            "headers": {
                "Authorization": f"Bearer {user['github_token']}" 
            }
        }
    })
    tools = await client.get_tools() # (3)
    
    # 您的工具调用逻辑
    
    tool_messages = ...
    return {"messages": tool_messages}
```

1. MCP 仅支持向 `streamable_http` 和 `sse` `transport` 服务器发出的请求添加标头。
2. 您的 MCP 服务器 URL。
3. 从您的 MCP 服务器获取可用工具。

_也可以通过[在运行时重建您的图](https://langchain-ai.github.io/langgraph/cloud/deployment/graph_rebuild/) 来为新运行配置不同的设置来完成此操作_

## 会话行为

当前的 LangGraph MCP 实现不支持会话。每个 `/mcp` 请求都是无状态且独立的。

## 身份验证

`/mcp` 端点使用与 LangGraph API 其余部分相同的身份验证。有关设置详细信息，请参阅[身份验证指南](./auth.md)。

## 禁用 MCP

要禁用 MCP 端点，请在 `langgraph.json` 配置文件中将 `disable_mcp` 设置为 `true`：

```json
{
  "http": {
    "disable_mcp": true
  }
}
```

这将阻止服务器公开 `/mcp` 端点。