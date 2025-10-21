# 指南

本节中的页面提供了以下主题的概念概述和操作方法：

## Agent 开发

- [概述](../agents/overview.md): 使用预构建的组件来构建 Agent。
- [运行 Agent](../agents/run_agents.md): 通过提供输入、解释输出、启用流式传输和控制执行限制来运行 Agent。

## LangGraph API

- [Graph API](../concepts/low_level.md): 使用 Graph API 以图（graph）范式定义工作流。
- [Functional API](../concepts/functional_api.md): 使用 Functional API 以函数式（functional）范式构建工作流，无需考虑图结构。
- [Runtime](../concepts/pregel.md): Pregel 实现了 LangGraph 的运行时，负责管理 LangGraph 应用程序的执行。

## 核心功能

这些功能在 LangGraph OSS 和 LangGraph 平台中都可用。

- [流式传输](../concepts/streaming.md): 从 LangGraph 图流式传输输出。
- [持久化](../concepts/persistence.md): 持久化 LangGraph 图的状态。
- [持久化执行](../concepts/durable_execution.md): 在图执行的关键节点保存进度。
- [内存](../concepts/memory.md): 记录先前交互的信息。
- [上下文](../agents/context.md): 将外部数据传递给 LangGraph 图，为图执行提供上下文。
- [模型](../agents/models.md): 将各种 LLM 集成到你的 LangGraph 应用程序中。
- [工具](../concepts/tools.md): 直接与外部系统进行交互。
- [人工干预（Human-in-the-loop）](../concepts/human_in_the_loop.md): 在工作流的任何节点暂停图并等待人工输入。
- [时间旅行](../concepts/time-travel.md): 回溯到 LangGraph 图执行的特定节点。
- [子图](../concepts/subgraphs.md): 构建模块化图。
- [多 Agent（Multi-agent）](../concepts/multi_agent.md): 将复杂的工作流分解为多个 Agent。
- [MCP](../concepts/mcp.md): 在 LangGraph 图中使用 MCP 服务器。
- [评估](../agents/evals.md): 使用 LangSmith 评估你的图的性能。