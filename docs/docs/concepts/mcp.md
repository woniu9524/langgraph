# MCP

[模型上下文协议 (Model Context Protocol, MCP)](https://modelcontextprotocol.io/introduction) 是一项开放的协议，它规范了应用程序如何向语言模型提供工具和上下文。LangGraph 代理可以通过 `langchain-mcp-adapters` 库使用 MCP 服务器上定义的工具。

![MCP](../agents/assets/mcp.png)

安装 `langchain-mcp-adapters` 库即可在 LangGraph 中使用 MCP 工具：

:::python
```bash
pip install langchain-mcp-adapters
```
:::

:::js
```bash
npm install @langchain/mcp-adapters
```
:::