---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 使用 MCP

[模型上下文协议 (MCP)](https://modelcontextprotocol.io/introduction) 是一项开放协议，用于标准化应用程序如何向语言模型提供工具和上下文。LangGraph 代理可以通过 `langchain-mcp-adapters` 库使用 MCP 服务器上定义的工具。

![MCP](./assets/mcp.png)

:::python
安装 `langchain-mcp-adapters` 库即可在 LangGraph 中使用 MCP 工具：

```bash
pip install langchain-mcp-adapters
```

:::

:::js
安装 `@langchain/mcp-adapters` 库即可在 LangGraph 中使用 MCP 工具：

```bash
npm install langchain-mcp-adapters
```

:::

## 使用 MCP 工具

:::python
`langchain-mcp-adapters` 包允许代理使用跨一个或多个 MCP 服务器定义的工具。

=== "在代理中"

    ```python title="使用 MCP 服务器上定义的工具的代理"
    # highlight-next-line
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    client = MultiServerMCPClient(
        {
            "math": {
                "command": "python",
                # 替换为 math_server.py 文件的绝对路径
                "args": ["/path/to/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # 确保在 8000 端口启动天气服务器
                "url": "http://localhost:8000/mcp",
                "transport": "streamable_http",
            }
        }
    )
    # highlight-next-line
    tools = await client.get_tools()
    agent = create_react_agent(
        "anthropic:claude-3-7-sonnet-latest",
        # highlight-next-line
        tools
    )
    math_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
    )
    weather_response = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
    )
    ```

=== "在工作流中"

    ```python title="在工作流中使用 MCP 工具及 ToolNode"
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, MessagesState, START, END
    from langgraph.prebuilt import ToolNode

    # 初始化模型
    model = init_chat_model("anthropic:claude-3-5-sonnet-latest")

    # 设置 MCP 客户端
    client = MultiServerMCPClient(
        {
            "math": {
                "command": "python",
                # 确保更新为 math_server.py 文件的完整绝对路径
                "args": ["./examples/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # 确保在 8000 端口启动天气服务器
                "url": "http://localhost:8000/mcp/",
                "transport": "streamable_http",
            }
        }
    )
    tools = await client.get_tools()

    # 将工具绑定到模型
    model_with_tools = model.bind_tools(tools)

    # 创建 ToolNode
    tool_node = ToolNode(tools)

    def should_continue(state: MessagesState):
        messages = state["messages"]
        last_message = messages[-1]
        if last_message.tool_calls:
            return "tools"
        return END

    # 定义 call_model 函数
    async def call_model(state: MessagesState):
        messages = state["messages"]
        response = await model_with_tools.ainvoke(messages)
        return {"messages": [response]}

    # 构建图
    builder = StateGraph(MessagesState)
    builder.add_node("call_model", call_model)
    builder.add_node("tools", tool_node)

    builder.add_edge(START, "call_model")
    builder.add_conditional_edges(
        "call_model",
        should_continue,
    )
    builder.add_edge("tools", "call_model")

    # 编译图
    graph = builder.compile()

    # 测试图
    math_response = await graph.ainvoke(
        {"messages": [{"role": "user", "content": "what's (3 + 5) x 12?"}]}
    )
    weather_response = await graph.ainvoke(
        {"messages": [{"role": "user", "content": "what is the weather in nyc?"}]}
    )
    ```

:::

:::js
`@langchain/mcp-adapters` 包允许代理使用跨一个或多个 MCP 服务器定义的工具。

=== "在代理中"

    ```typescript title="使用 MCP 服务器上定义的工具的代理"
    // highlight-next-line
    import { MultiServerMCPClient } from "langchain-mcp-adapters/client";
    import { ChatAnthropic } from "@langchain/langgraph/prebuilt";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    // highlight-next-line
    const client = new MultiServerMCPClient({
      math: {
        command: "node",
        // 替换为 math_server.js 文件的绝对路径
        args: ["/path/to/math_server.js"],
        transport: "stdio",
      },
      weather: {
        // 确保在 8000 端口启动天气服务器
        url: "http://localhost:8000/mcp",
        transport: "streamable_http",
      },
    });

    // highlight-next-line
    const tools = await client.getTools();
    const agent = createReactAgent({
      llm: new ChatAnthropic({ model: "claude-3-7-sonnet-latest" }),
      // highlight-next-line
      tools,
    });

    const mathResponse = await agent.invoke({
      messages: [{ role: "user", content: "what's (3 + 5) x 12?" }],
    });

    const weatherResponse = await agent.invoke({
      messages: [{ role: "user", content: "what is the weather in nyc?" }],
    });
    ```

=== "在工作流中"

    ```typescript
    import { MultiServerMCPClient } from "langchain-mcp-adapters/client";
    import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
    import { ToolNode } from "@langchain/langgraph/prebuilt";
    import { ChatOpenAI } from "@langchain/openai";
    import { AIMessage } from "@langchain/core/messages";
    import { z } from "zod";

    const model = new ChatOpenAI({ model: "gpt-4" });

    const client = new MultiServerMCPClient({
      math: {
        command: "node",
        // 确保更新为 math_server.js 文件的完整绝对路径
        args: ["./examples/math_server.js"],
        transport: "stdio",
      },
      weather: {
        // 确保在 8000 端口启动天气服务器
        url: "http://localhost:8000/mcp/",
        transport: "streamable_http",
      },
    });

    const tools = await client.getTools();

    const builder = new StateGraph(MessagesZodState)
      .addNode("callModel", async (state) => {
        const response = await model.bindTools(tools).invoke(state.messages);
        return { messages: [response] };
      })
      .addNode("tools", new ToolNode(tools))
      .addEdge(START, "callModel")
      .addConditionalEdges("callModel", (state) => {
        const lastMessage = state.messages.at(-1) as AIMessage | undefined;
        if (!lastMessage?.tool_calls?.length) {
          return "__end__";
        }
        return "tools";
      })
      .addEdge("tools", "callModel");

    const graph = builder.compile();

    const mathResponse = await graph.invoke({
      messages: [{ role: "user", content: "what's (3 + 5) x 12?" }],
    });

    const weatherResponse = await graph.invoke({
      messages: [{ role: "user", content: "what is the weather in nyc?" }],
    });
    ```

:::

## 自定义 MCP 服务器

:::python
要创建自己的 MCP 服务器，可以使用 `mcp` 库。该库提供了一种简单的方法来定义工具并将其作为服务器运行。

安装 MCP 库：

```bash
pip install mcp
```

:::

:::js
要创建自己的 MCP 服务器，可以使用 `@modelcontextprotocol/sdk` 库。该库提供了一种简单的方法来定义工具并将其作为服务器运行。

安装 MCP SDK：

```bash
npm install @modelcontextprotocol/sdk
```

:::

使用以下参考实现来测试你的代理与 MCP 工具服务器的集成。

:::python

```python title="示例数学服务器 (stdio 传输)"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """加两个数"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """乘两个数"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

:::

:::js

```typescript title="示例数学服务器 (stdio 传输)"
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { StdioServerTransport } from "@modelcontextprotocol/sdk/server/stdio.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";

const server = new Server(
  {
    name: "math-server",
    version: "0.1.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "add",
        description: "加两个数",
        inputSchema: {
          type: "object",
          properties: {
            a: {
              type: "number",
              description: "第一个数",
            },
            b: {
              type: "number",
              description: "第二个数",
            },
          },
          required: ["a", "b"],
        },
      },
      {
        name: "multiply",
        description: "乘两个数",
        inputSchema: {
          type: "object",
          properties: {
            a: {
              type: "number",
              description: "第一个数",
            },
            b: {
              type: "number",
              description: "第二个数",
            },
          },
          required: ["a", "b"],
        },
      },
    ],
  };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  switch (request.params.name) {
    case "add": {
      const { a, b } = request.params.arguments as { a: number; b: number };
      return {
        content: [
          {
            type: "text",
            text: String(a + b),
          },
        ],
      };
    }
    case "multiply": {
      const { a, b } = request.params.arguments as { a: number; b: number };
      return {
        content: [
          {
            type: "text",
            text: String(a * b),
          },
        ],
      };
    }
    default:
      throw new Error(`未知的工具: ${request.params.name}`);
  }
});

async function main() {
  const transport = new StdioServerTransport();
  await server.connect(transport);
  console.error("Math MCP 服务器正在通过 stdio 运行");
}

main();
```

:::

:::python

```python title="示例天气服务器 (Streamable HTTP 传输)"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Weather")

@mcp.tool()
async def get_weather(location: str) -> str:
    """获取指定地点的天气。"""
    return "It's always sunny in New York"

if __name__ == "__main__":
    mcp.run(transport="streamable-http")
```

:::

:::js

```typescript title="示例天气服务器 (HTTP 传输)"
import { Server } from "@modelcontextprotocol/sdk/server/index.js";
import { SSEServerTransport } from "@modelcontextprotocol/sdk/server/sse.js";
import {
  CallToolRequestSchema,
  ListToolsRequestSchema,
} from "@modelcontextprotocol/sdk/types.js";
import express from "express";

const app = express();
app.use(express.json());

const server = new Server(
  {
    name: "weather-server",
    version: "0.1.0",
  },
  {
    capabilities: {
      tools: {},
    },
  }
);

server.setRequestHandler(ListToolsRequestSchema, async () => {
  return {
    tools: [
      {
        name: "get_weather",
        description: "获取指定地点的天气",
        inputSchema: {
          type: "object",
          properties: {
            location: {
              type: "string",
              description: "获取天气信息的地点",
            },
          },
          required: ["location"],
        },
      },
    ],
  };
});

server.setRequestHandler(CallToolRequestSchema, async (request) => {
  switch (request.params.name) {
    case "get_weather": {
      const { location } = request.params.arguments as { location: string };
      return {
        content: [
          {
            type: "text",
            text: `It's always sunny in ${location}`,
          },
        ],
      };
    }
    default:
      throw new Error(`未知的工具: ${request.params.name}`);
  }
});

app.post("/mcp", async (req, res) => {
  const transport = new SSEServerTransport("/mcp", res);
  await server.connect(transport);
});

const PORT = process.env.PORT || 8000;
app.listen(PORT, () => {
  console.log(`Weather MCP 服务器正在端口 ${PORT} 上运行`);
});
```

:::

:::python

## 更多资源

- [MCP 文档](https://modelcontextprotocol.io/introduction)
- [MCP 传输文档](https://modelcontextprotocol.io/docs/concepts/transports)
- [langchain_mcp_adapters](https://github.com/langchain-ai/langchain-mcp-adapters)
  :::

:::js

## 更多资源

- [MCP 文档](https://modelcontextprotocol.io/introduction)
- [MCP 传输文档](https://modelcontextprotocol.io/docs/concepts/transports)
- [`@langchain/mcp-adapters`](https://npmjs.com/package/@langchain/mcp-adapters)
  :::