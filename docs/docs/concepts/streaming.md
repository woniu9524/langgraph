---
search:
boost: 2
---

# Streaming

LangGraph 实现了一个流式系统，用于呈现实时更新，从而提供响应迅速且透明的用户体验。

LangGraph 的流式系统可将图运行的实时反馈呈现给您的应用程序。
您可以流式传输的数据主要有三个类别：

1. **工作流进度** — 在每个图节点执行后获取状态更新。
2. **LLM token** — 在生成时流式传输语言模型 token。
3. **自定义更新** — 发出用户定义的信号（例如，“已获取 10/100 条记录”）。

## 使用 LangGraph 流式传输可以实现的功能

- [**流式传输 LLM token**](../how-tos/streaming.md#messages) — 从任何地方捕获 token 流：节点内部、子图或工具。
- [**从工具发出进度通知**](../how-tos/streaming.md#stream-custom-data) — 直接从工具函数发送自定义更新或进度信号。
- [**从子图流式传输**](../how-tos/streaming.md#stream-subgraph-outputs) — 包括来自父图和任何嵌套子图的输出。
- [**使用任何 LLM**](../how-tos/streaming.md#use-with-any-llm) — 流式传输任何 LLM 的 token，即使它不是 LangChain 模型，也可使用 `custom` 流式传输模式。
- [**使用多种流式传输模式**](../how-tos/streaming.md#stream-multiple-modes) — 可以选择 `values`（完整状态）、`updates`（状态增量）、`messages`（LLM token + 元数据）、`custom`（任意用户数据），或 `debug`（详细跟踪）。