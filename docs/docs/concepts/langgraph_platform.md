---
search:
  boost: 2
---

# LangGraph 平台

利用 **LangGraph 平台** 开发、部署、扩展和管理代理 — 这是专为长期运行的代理工作流而构建的平台。

!!! tip "开始使用 LangGraph 平台"

    请查看 [LangGraph 平台快速入门指南](../tutorials/langgraph-platform/local-server.md)，了解如何使用 LangGraph 平台在本地运行 LangGraph 应用程序的说明。

## 为什么使用 LangGraph 平台？

<div align="center"><iframe width="560" height="315" src="https://www.youtube.com/embed/pfAQxBS5z88?si=XGS6Chydn6lhSO1S" title="What is LangGraph Platform?" frameborder="0" allow="accelerometer; autoplay; clipboard-write; encrypted-media; gyroscope; picture-in-picture; web-share" referrerpolicy="strict-origin-when-cross-origin" allowfullscreen></iframe></div>

LangGraph 平台让您的代理（无论是使用 LangGraph 还是其他框架构建的）轻松投入生产 — 这样您就可以专注于您的应用程序逻辑，而不是基础设施。一键部署即可获得实时端点，并使用我们强大的 API 和内置任务队列来处理生产规模。

- **[流式支持](../cloud/how-tos/streaming.md)**：随着代理变得越来越复杂，它们通常受益于将令牌输出和中间状态流式传输回用户。没有它，用户将不得不等待可能很长的操作而没有任何反馈。LangGraph 服务器提供多种针对各种应用程序需求优化的流式传输模式。

- **[后台运行](../cloud/how-tos/background_run.md)**：对于处理时间较长的代理（例如数小时），保持连接处于打开状态可能不切实际。LangGraph 服务器支持在后台启动代理运行，并提供轮询端点和 Webhook 来有效监控运行状态。
 
- 支持长期运行：常规服务器设置在处理耗时较长的请求时经常会遇到超时或中断。LangGraph 服务器的 API 通过发送常规心跳信号来为这些任务提供强大的支持，防止在长时间处理过程中意外断开连接。

- 处理突发流量：某些应用程序，特别是那些具有实时用户交互的应用程序，可能会遇到“突发”请求负载，即大量请求同时命中服务器。LangGraph 服务器包含一个任务队列，确保即使在高负载下，请求也能得到一致处理而不丢失。

- **[双重短信](../cloud/how-tos/interrupt_concurrent.md)**：在用户驱动的应用程序中，用户快速发送多条消息是很常见的。如果不妥善处理，“双重短信”可能会破坏代理流程。LangGraph 服务器提供了内置策略来解决和管理此类交互。

- **[检查点和内存管理](persistence.md#checkpoints)**：对于需要持久性的代理（例如会话记忆），部署一个健壮的存储解决方案可能很复杂。LangGraph 平台包括优化的 [检查点](persistence.md#checkpoints) 和 [内存存储](persistence.md#memory-store)，可以在会话之间管理状态，而无需自定义解决方案。

- **[人工干预支持](../cloud/how-tos/human_in_the_loop_breakpoint.md)**：在许多应用程序中，用户需要一种方法来干预代理进程。LangGraph 服务器为人工干预场景提供了专用端点，简化了将手动监督集成到代理工作流中的过程。

- **[LangGraph Studio](./langgraph_studio.md)**：支持实现 LangGraph 服务器 API 协议的代理系统的可视化、交互和调试。Studio 还与 LangSmith 集成，支持跟踪、评估和提示工程。

- **[部署](./deployment_options.md)**：LangGraph 平台提供四种部署方式：[Cloud SaaS](../concepts/langgraph_cloud.md)、[Self-Hosted Data Plane](../concepts/langgraph_self_hosted_data_plane.md)、[Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) 和 [Standalone Container](../concepts/langgraph_standalone_container.md)。