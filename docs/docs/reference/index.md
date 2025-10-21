---
title: 参考
description: LangGraph 的 API 参考
search:
  boost: 0.5
---

<style>
.md-sidebar {
  display: block !important;
}
</style>

# 参考

欢迎来到 LangGraph 参考文档！这些页面详细介绍了您在使用 LangGraph 构建时将使用的核心接口。每个部分涵盖了生态系统的不同部分。

!!! tip

    如果您是初学者，请参阅 [LangGraph 基础知识](../concepts/why-langgraph.md) 了解主要概念和使用模式的介绍。


## LangGraph

LangGraph 开源库的核心 API。

- [Graphs](graphs.md)：主图抽象和用法。
- [Functional API](func.md)：图的函数式编程接口。
- [Pregel](pregel.md)：受 Pregel 启发的计算模型。
- [Checkpointing](checkpoints.md)：保存和恢复图状态。
- [Storage](store.md)：存储后端和选项。
- [Caching](cache.md)：用于性能的缓存机制。
- [Types](types.md)：图组件的类型定义。
- [Config](config.md)：配置选项。
- [Errors](errors.md)：错误类型和处理。
- [Constants](constants.md)：全局常量。
- [Channels](channels.md)：消息传递和通道。

## 预构建组件

用于常见工作流、代理和其他模式的高级抽象。

- [Agents](agents.md)：内置代理模式。
- [Supervisor](supervisor.md)：编排和委托。
- [Swarm](swarm.md)：多代理协作。
- [MCP Adapters](mcp.md)：与外部系统的集成。

## LangGraph 平台

用于部署和连接到 LangGraph 平台的工具。

- [SDK (Python)](../cloud/reference/sdk/python_sdk_ref.md)：用于与 LangGraph Server 实例交互的 Python SDK。
- [SDK (JS/TS)](../cloud/reference/sdk/js_ts_sdk_ref.md)：用于与 LangGraph Server 实例交互的 JavaScript/TypeScript SDK。
- [RemoteGraph](remote_graph.md)：用于连接到 LangGraph Server 实例的 `Pregel` 抽象。

有关更多参考文档，请参阅 [LangGraph 平台参考](https://docs.langchain.com/langgraph-platform/reference-overview)。