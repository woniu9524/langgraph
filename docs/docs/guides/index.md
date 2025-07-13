# 指南

本节的页面提供了以下主题的概念概述和操作指南：

## LangGraph API

- [Graph API](../concepts/low_level.md): 使用 Graph API 通过图范式定义工作流。
- [Functional API](../concepts/functional_api.md): 使用 Functional API 通过函数式范式构建工作流，无需考虑图结构。
- [Runtime](../concepts/pregel.md): Pregel 实现了 LangGraph 的运行时，负责管理 LangGraph 应用程序的执行。

## 核心功能

这些功能在 LangGraph OSS 和 LangGraph Platform 中均可用。

- [Streaming](../concepts/streaming.md): 从 LangGraph 图流式传输输出。
- [Persistence](../concepts/persistence.md): 持久化 LangGraph 图的状态。
- [Durable execution](../concepts/durable_execution.md): 在图执行的关键节点保存进度。
- [Memory](../concepts/memory.md): 记忆先前交互的信息。
- [Context](../agents/context.md): 将外部数据传递给 LangGraph 图，为图执行提供上下文。
- [Models](../agents/models.md): 将各种 LLM 集成到你的 LangGraph 应用程序中。
- [Tools](../concepts/tools.md): 直接与外部系统交互。
- [Human-in-the-loop](../concepts/human_in_the_loop.md): 在工作流的任何节点暂停图并等待用户输入。
- [Time travel](../concepts/time-travel.md): 追溯到 LangGraph 图执行的特定节点。
- [Subgraphs](../concepts/subgraphs.md): 构建模块化图。
- [Multi-agent](../concepts/multi_agent.md): 将复杂工作流分解为多个代理。
- [MCP](../concepts/mcp.md): 在 LangGraph 图中使用 MCP 服务器。
- [Evaluation](../agents/evals.md): 使用 LangSmith 评估你的图的性能。

## 仅平台功能

这些功能仅在 [LangGraph Platform](../concepts/langgraph_platform.md) 中可用。

- [Authentication and access control](../concepts/auth.md): 对用户进行身份验证和授权，以访问 Langraph 图。
- [Assistants](../concepts/assistants.md): 构建可用于与 LangGraph 图交互的助手。
- [Double-texting](../concepts/double_texting.md): 在 LangGraph 图中处理双重文本（在返回首次响应之前的连续消息）。
- [Webhooks](../cloud/concepts/webhooks.md): 向 LangGraph 图发送 Webhook。
- [Cron jobs](../cloud/concepts/cron_jobs.md): 安排作业在特定时间运行。
- [Server customization](../how-tos/http/custom_lifespan.md): 自定义运行 LangGraph 图的服务器。
- [Data management](../cloud/concepts/data_storage_and_privacy.md): 管理 LangGraph 图中的数据。
- [Deployment](../concepts/deployment_options.md): 将 LangGraph 图部署到服务器。