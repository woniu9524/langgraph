---
tags:
  - mcp
  - platform
hide:
  - tags
---

# LangGraph Server 中的 MCP 端点

[模型上下文协议 (MCP)](./mcp.md) 是一种开放协议，用于以与模型无关的格式描述工具和数据源，使 LLM 能够通过结构化 API 进行发现和使用。

[LangGraph Server](./langgraph_server.md) 使用 [Streamable HTTP 传输](https://spec.modelcontextprotocol.io/specification/2025-03-26/basic/transports/#streamable-http) 来实现 MCP。这允许将 LangGraph **代理（agents）** 暴露为 **MCP 工具**，使其可用于任何支持 Streamable HTTP 的符合 MCP 标准的客户端。

MCP 端点位于 [LangGraph Server](./langgraph_server.md) 的 `/mcp` 路径下。

## 要求

:::python
要使用 MCP，请确保已安装以下依赖项：

- `langgraph-api >= 0.2.3`
- `langgraph-sdk >= 0.1.61`

运行以下命令进行安装：

```bash
pip install "langgraph-api>=0.2.3" "langgraph-sdk>=0.1.61"
```

:::

:::js
要使用 MCP，请确保已同时安装 api 和 sdk 包。

```bash
npm install @langchain/langgraph-api @langchain/langgraph-sdk
```

:::

## 将代理暴露为 MCP 工具

部署后，您的代理将出现在 MCP 端点中，配置如下：

- **工具名称（Tool name）**: 代理的名称。
- **工具描述（Tool description）**: 代理的描述。
- **工具输入模式（Tool input schema）**: 代理的输入模式。

### 设置名称和描述

您可以在 `langgraph.json` 中设置代理的名称和描述：

:::python

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

:::
:::js

```json
{
  "graphs": {
    "my_agent": {
      "path": "./my_agent/agent.ts:graph",
      "description": "A description of what the agent does"
    }
  },
  "env": ".env"
}
```

:::

部署后，您可以使用 LangGraph SDK 更新名称和描述。

### Schema

定义清晰、最小化的输入和输出模式，以避免向 LLM 暴露不必要的内部复杂性。

:::python
默认的 [MessagesState](./low_level.md#messagesstate) 使用 `AnyMessage`，它支持多种消息类型，但对于直接暴露给 LLM 来说过于通用。
:::

相反，请定义使用显式类型化输入和输出结构的**自定义代理或工作流**。

例如，一个回答文档问题的 Worfklow 可能如下所示：

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 定义输入模式
class InputState(TypedDict):
    question: str

# 定义输出模式
class OutputState(TypedDict):
    answer: str

# 组合输入和输出
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

## 用法概览

要启用 MCP：

- 升级至 langgraph-api>=0.2.3。如果您正在部署 LangGraph Platform，在创建新修订版本时会自动完成。
- MCP 工具（代理）将自动暴露。
- 连接到任何支持 Streamable HTTP 的符合 MCP 标准的客户端。

### 客户端

:::python
使用符合 MCP 标准的客户端连接到 LangGraph 服务器。以下示例展示了如何使用 [langchain-mcp-adapters](https://github.com/langchain-ai/langchain-mcp-adapters) 进行连接。

通过以下命令安装适配器：

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

            # 将远程图加载为工具
            tools = await load_mcp_tools(session)

            # 使用工具创建并运行一个 react agent
            agent = create_react_agent("openai:gpt-4.1", tools)

            # 使用消息调用代理
            agent_response = await agent.ainvoke({"messages": "What can the finance agent do for me?"})
            print(agent_response)

if __name__ == "__main__":
    asyncio.run(main())
```

:::

:::js
使用符合 MCP 标准的客户端连接到 LangGraph 服务器。以下示例展示了如何使用 [`@langchain/mcp-adapters`](https://npmjs.com/package/@langchain/mcp-adapters) 进行连接。

```bash
npm install @langchain/mcp-adapters
```

以下是如何连接到远程 MCP 端点并使用代理作为工具的示例：

```typescript
import { MultiServerMCPClient } from "@langchain/mcp-adapters";
import { createReactAgent } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";

async function main() {
  const client = new MultiServerMCPClient({
    mcpServers: {
      "finance-agent": {
        url: "https://mcp-finance-agent.xxx.us.langgraph.app/mcp",
        headers: {
          "X-Api-Key": "lsv2_pt_your_api_key",
        },
      },
    },
  });

  const tools = await client.getTools();

  const model = new ChatOpenAI({
    model: "gpt-4o-mini",
    temperature: 0,
  });

  const agent = createReactAgent({
    model,
    tools,
  });

  const response = await agent.invoke({
    input: "What can the finance agent do for me?",
  });

  console.log(response);
}

main();
```

:::

## Session 行为

当前的 LangGraph MCP 实现不支持会话。每个 `/mcp` 请求都是无状态且独立的。

## 认证

`/mcp` 端点使用与 LangGraph API 其余部分相同的认证方式。有关设置详情，请参阅 [认证指南](./auth.md)。

## 禁用 MCP

要禁用 MCP 端点，请在 `langgraph.json` 配置文件中将 `disable_mcp` 设置为 `true`：

```json
{
  "http": {
    "disable_mcp": true
  }
}
```

这将阻止服务器暴露 `/mcp` 端点。