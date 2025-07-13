---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 使用 MCP

模型上下文协议 (MCP) 是一种开放协议，它标准化了应用程序如何向语言模型提供工具和上下文。LangGraph 代理可以通过 `langchain-mcp-adapters` 库使用定义在 MCP 服务器上的工具。

## 使用 MCP 工具

`langchain-mcp-adapters` 包使代理能够使用在一个或多个 MCP 服务器上定义的工具。

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
                # 替换为指向你的 math_server.py 文件的绝对路径
                "args": ["/path/to/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # 确保在端口 8000 上启动你的天气服务器
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

    ```python
    from langchain_mcp_adapters.client import MultiServerMCPClient
    from langgraph.graph import StateGraph, MessagesState, START
    from langgraph.prebuilt import ToolNode, tools_condition

    from langchain.chat_models import init_chat_model
    model = init_chat_model("openai:gpt-4.1")

    client = MultiServerMCPClient(
        {
            "math": {
                "command": "python",
                # 确保更新为指向你的 math_server.py 文件的完整绝对路径
                "args": ["./examples/math_server.py"],
                "transport": "stdio",
            },
            "weather": {
                # 确保在端口 8000 上启动你的天气服务器
                "url": "http://localhost:8000/mcp/",
                "transport": "streamable_http",
            }
        }
    )
    tools = await client.get_tools()

    def call_model(state: MessagesState):
        response = model.bind_tools(tools).invoke(state["messages"])
        return {"messages": response}

    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_node(ToolNode(tools))
    builder.add_edge(START, "call_model")
    builder.add_conditional_edges(
        "call_model",
        tools_condition,
    )
    builder.add_edge("tools", "call_model")
    graph = builder.compile()
    math_response = await graph.ainvoke({"messages": "what's (3 + 5) x 12?"})
    weather_response = await graph.ainvoke({"messages": "what is the weather in nyc?"})
    ```



## 自定义 MCP 服务器

要创建自己的 MCP 服务器，你可以使用 `mcp` 库。该库提供了一种简单的方法来定义工具并将其作为服务器运行。

安装 MCP 库：

```bash
pip install mcp
```
使用以下参考实现来测试你的代理与 MCP 工具服务器的集成。

```python title="示例数学服务器 (stdio 传输)"
from mcp.server.fastmcp import FastMCP

mcp = FastMCP("Math")

@mcp.tool()
def add(a: int, b: int) -> int:
    """将两个数字相加"""
    return a + b

@mcp.tool()
def multiply(a: int, b: int) -> int:
    """将两个数字相乘"""
    return a * b

if __name__ == "__main__":
    mcp.run(transport="stdio")
```

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

## 附加资源

- [MCP 文档](https://modelcontextprotocol.io/introduction)
- [MCP 传输文档](https://modelcontextprotocol.io/docs/concepts/transports)
- [langchain_mcp_adapters](https://github.com/langchain-ai/langchain-mcp-adapters)