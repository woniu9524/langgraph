---
search:
  boost: 2
---

# FAQ

常见问题及解答！

## 我需要使用 LangChain 才能使用 LangGraph 吗？有什么区别？

不需要。LangGraph 是一个用于复杂代理系统的编排框架，它比 LangChain 代理更底层，并且可控性更强。LangChain 提供了一个标准的接口来与模型和其他组件进行交互，适用于直接的链式调用和检索流程。

## LangGraph 与其他代理框架有何不同？

其他代理框架可以处理简单、通用的任务，但在复杂任务面前却力不从心。LangGraph 提供了一个更具表现力的框架来处理您独特的任务，而不会将您限制在单一的黑盒认知架构中。

## LangGraph 会影响我的应用性能吗？

LangGraph 不会为您的代码增加任何额外开销，并且是专门为流式处理工作流而设计的。

## LangGraph 是开源的吗？免费吗？

是的。LangGraph 是一个采用 MIT 许可的开源库，可免费使用。

## LangGraph 和 LangGraph Platform 有何不同？

LangGraph 是一个有状态的编排框架，为代理工作流增加了额外的控制力。LangGraph Platform 是一个用于部署和扩展 LangGraph 应用程序的服务，它提供了一个用于构建代理用户体验的规范化 API，并集成了开发者工作室。

| 特征            | LangGraph (开源)                                     | LangGraph Platform                                                                       |
| --------------- | ---------------------------------------------------- | ---------------------------------------------------------------------------------------- |
| 描述            | 有状态的代理应用程序编排框架                         | 用于部署 LangGraph 应用程序的可扩展基础设施                                              |
| SDK             | Python 和 JavaScript                                 | Python 和 JavaScript                                                                     |
| HTTP API        | 无                                                   | 是——可用于检索和更新状态或长期记忆，或创建可配置的助手                                   |
| 流式处理        | 基本                                                 | 用于逐字消息的专用模式                                                                   |
| Checkpointer    | 社区贡献                                             | 开箱即用支持                                                                             |
| 持久化层        | 自我管理                                             | 托管的 Postgres，具有高效的存储                                                          |
| 部署            | 自我管理                                             | • 云 SaaS <br> • 免费自托管 <br> • 企业版（付费自托管）                                  |
| 可扩展性        | 自我管理                                             | 任务队列和服务器的自动扩展                                                               |
| 容错性          | 自我管理                                             | 自动重试                                                                                 |
| 并发控制        | 简单的线程                                           | 支持双重文本输入（"double-texting"——此处可能指一种同步机制，原文此处表述可能不够清晰）。 |
| 调度            | 无                                                   | Cron 调度                                                                                |
| 监控            | 无                                                   | 与 LangSmith 集成，用于可观测性                                                          |
| IDE 集成        | LangGraph Studio                                     | LangGraph Studio                                                                         |

## LangGraph Platform 是开源的吗？

不是。LangGraph Platform 是专有软件。

提供了一个免费的自托管版本 LangGraph Platform，可以使用基本功能。云 SaaS 部署选项和自托管部署选项是付费服务。[联系我们的销售团队](https://www.langchain.com/contact-sales)了解更多信息。

有关更多信息，请参阅我们的 [LangGraph Platform 定价页面](https://www.langchain.com/pricing-langgraph-platform)。

## LangGraph 是否支持不支持工具调用的 LLM？

是的！您可以使用 LangGraph 和任何 LLM。我们主要使用支持工具调用的 LLM 是因为这通常是让 LLM 决定下一步操作最方便的方式。如果您的 LLM 不支持工具调用，您仍然可以使用它——您只需要编写一些逻辑，将原始 LLM 字符串响应转换为下一步操作的决策。

## LangGraph 是否支持开源 LLM？

是的！LangGraph 完全不区分底层使用的 LLM。我们在大多数教程中使用闭源 LLM 的主要原因是它们无缝支持工具调用，而许多开源 LLM 并不支持。但工具调用并非必需（参见 [本节](#does-langgraph-work-with-llms-that-dont-support-tool-calling)），因此您完全可以使用 LangGraph 和开源 LLM。

## 我可以在不登录 LangSmith 的情况下使用 LangGraph Studio 吗？

是的！您可以使用 [LangGraph Server 的开发版本](../tutorials/langgraph-platform/local-server.md) 在本地运行后端。
这将连接到作为 LangSmith 一部分托管的前端 Studio。
如果您设置了环境变量 `LANGSMITH_TRACING=false`，则不会将任何 trace 发送到 LangSmith。

## LangGraph Platform 使用中的“已执行节点”是什么意思？

**已执行节点 (Nodes Executed)** 是 LangGraph 应用程序在调用期间被调用并成功完成的节点的总数。如果在执行期间某个节点未被调用，或者该节点最终处于错误状态，则这些节点不会被计算在内。如果一个节点被调用并成功完成多次，每次执行都会被计算在内。