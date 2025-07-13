---
search:
  boost: 2
---

# FAQ

常见问题解答！

## 我需要使用 LangChain 才能使用 LangGraph 吗？有什么区别？

不需要。LangGraph 是一个用于复杂代理系统的编排框架，它比 LangChain 代理更底层、更可控。LangChain 提供了一个标准的接口来与模型和其他组件进行交互，这对于直接的链式调用和检索流程非常有用。

## LangGraph 与其他代理框架有何不同？

其他代理框架可以胜任简单、通用的任务，但在处理公司特定需求的复杂任务时则显得力不从心。LangGraph 提供了一个更具表现力的框架来处理公司独特的任务，而不会将用户限制在单一的黑盒认知架构中。

## LangGraph 会影响我的应用程序性能吗？

LangGraph 不会增加您的代码的任何开销，并且专门针对流式工作流程进行了设计。

## LangGraph 是开源的吗？免费吗？

是的。LangGraph 是一个 MIT 许可的开源库，可以免费使用。

## LangGraph 和 LangGraph Platform 有什么区别？

LangGraph 是一个有状态的编排框架，为代理工作流程带来了额外的控制。LangGraph Platform 是一个用于部署和扩展 LangGraph 应用程序的服务，提供了一个用于构建代理用户体验的标准化 API，并集成了开发者工作室。

| 功能            | LangGraph (开源)                                             | LangGraph Platform                                                                                     |
|-----------------|--------------------------------------------------------------|--------------------------------------------------------------------------------------------------------|
| 描述            | 有状态的代理应用程序编排框架                                     | 用于部署 LangGraph 应用程序的可扩展基础设施                                                             |
| SDK           | Python 和 JavaScript                                         | Python 和 JavaScript                                                                                   |
| HTTP API       | 无                                                           | 有 - 可用于检索和更新状态或长期记忆，或创建可配置的助手                                                       |
| 流式传输       | 基本                                                         | 用于逐令牌消息的专用模式                                                                                 |
| 检查点       | 社区贡献                                                     | 开箱即用支持                                                                                         |
| 持久化层      | 自我管理                                                     | 托管的 Postgres，具有高效的存储                                                                          |
| 部署          | 自我管理                                                     | • 云 SaaS <br> • 免费自托管 <br> • 企业版（付费自托管）                                                   |
| 可扩展性       | 自我管理                                                     | 任务队列和服务器的自动扩展                                                                               |
| 容错性       | 自我管理                                                     | 自动重试                                                                                             |
| 并发控制       | 简单的线程                                                   | 支持双重文本                                                                                         |
| 调度          | 无                                                           | Cron 调度                                                                                            |
| 监控          | 无                                                           | 与 LangSmith 集成，用于可观察性                                                                          |
| IDE 集成       | LangGraph Studio                                             | LangGraph Studio                                                                                       |

## LangGraph Platform 是开源的吗？

不是。LangGraph Platform 是专有软件。

有一个免费的自托管版本 LangGraph Platform，提供基本功能。云 SaaS 部署选项和自托管部署选项是付费服务。[联系我们的销售团队](https://www.langchain.com/contact-sales)了解更多信息。

有关更多信息，请参阅我们的 [LangGraph Platform 定价页面](https://www.langchain.com/pricing-langgraph-platform)。

## LangGraph 是否适用于不支持工具调用的 LLM？

是的！您可以使用 LangGraph 和任何 LLM。我们使用支持工具调用的 LLM 的主要原因是，这通常是让 LLM 决定做什么的最便捷方式。如果您的 LLM 不支持工具调用，您仍然可以使用它 - 您只需要编写一些逻辑，将原始的 LLM 字符串响应转换为有关要做什么的决策。

## LangGraph 是否兼容 OSS LLM？

是的！LangGraph 对底层使用的 LLM 完全没有偏好。我们在大多数教程中使用闭源 LLM 的主要原因是它们无缝支持工具调用，而 OSS LLM 通常不支持。但工具调用并非必需（请参阅 [本节](#does-langgraph-work-with-llms-that-dont-support-tool-calling)），因此您完全可以使用 LangGraph 和 OSS LLM。

## 我可以在不登录 LangSmith 的情况下使用 LangGraph Studio 吗？

是的！您可以使用 [LangGraph Server 的开发版本](../tutorials/langgraph-platform/local-server.md)在本地运行后端。
这将连接到作为 LangSmith 一部分的 Studio 前端。
如果您设置了环境变量 `LANGSMITH_TRACING=false`，则不会将任何跟踪发送到 LangSmith。

## “已执行节点”对 LangGraph Platform 的使用意味着什么？

**已执行节点 (Nodes Executed)** 是 LangGraph 应用程序中在应用程序调用期间被调用并成功完成的节点的聚合数量。如果在执行期间未调用图中的节点或节点以错误状态结束，则这些节点将不被计算在内。如果一个节点被调用并成功完成多次，每次调用都将被计算在内。