---
search:
  boost: 2
---

# LangGraph Studio

!!! info "先决条件"

    - [LangGraph Platform](./langgraph_platform.md)
    - [LangGraph Server](./langgraph_server.md)
    - [LangGraph CLI](./langgraph_cli.md)

LangGraph Studio 是一个专门的代理 IDE，可实现实现 LangGraph Server API 协议的代理系统的可视化、交互和调试。 Studio 还与 LangSmith 集成，以实现跟踪、评估和提示工程。

![](img/lg_studio.png)

## 功能

LangGraph Studio 的主要功能：

- 可视化您的图架构
- [运行并与您的代理交互](../cloud/how-tos/invoke_studio.md)
- [管理助手](../cloud/how-tos/studio/manage_assistants.md)
- [管理线程](../cloud/how-tos/threads_studio.md)
- [迭代提示](../cloud/how-tos/iterate_graph_studio.md)
- [针对数据集运行实验](../cloud/how-tos/studio/run_evals.md)
- 管理[长期记忆](memory.md)
- 通过[时间旅行](time-travel.md)调试代理状态

LangGraph Studio 可用于部署在[LangGraph Platform](../cloud/quick_start.md)上的图，或通过[LangGraph Server](../tutorials/langgraph-platform/local-server.md)在本地运行的图。

Studio 支持两种模式：

### 图模式

图模式公开了 Studio 的完整功能集，特别适合您想要了解代理执行的尽可能多的细节时，包括遍历的节点、中间状态以及 LangSmith 集成（例如添加到数据集和 Playground）。

### 聊天模式

聊天模式是用于迭代和测试聊天特定代理的更简洁的用户界面。它对于业务用户和希望测试整体代理行为的用户非常有用。聊天模式仅支持状态包含或扩展了 [`MessagesState`](https://langchain-ai.github.io/langgraph/how-tos/graph-api/#messagesstate) 的图。

## 了解更多

- 请参阅此指南，了解如何[开始使用](../cloud/how-tos/studio/quick_start.md) LangGraph Studio。