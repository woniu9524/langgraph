---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# UI

您可以使用预先构建的聊天 UI，通过 [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 与任何 LangGraph 代理进行交互。使用 [部署版本](https://agentchat.vercel.app) 是最快的入门方式，它允许您与本地和已部署的 Graph 进行交互。

## 在 UI 中运行代理

首先，在本地设置 LangGraph API 服务器 [（请参阅本地服务器教程）](../tutorials/langgraph-platform/local-server.md) 或将您的代理部署到 [LangGraph Platform](https://langchain-ai.github.io/langgraph/cloud/quick_start/)。

然后，导航到 [Agent Chat UI](https://agentchat.vercel.app)，或者克隆仓库并 [在本地运行开发服务器](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#setup)：

<video controls src="../assets/base-chat-ui.mp4" type="video/mp4"></video>

!!! Tip

    UI 对渲染工具调用和工具结果消息开箱即用。要自定义显示哪些消息，请参阅 Agent Chat UI 文档中的 [隐藏聊天中的消息](https://github.com/langchain-ai/agent-chat-ui?tab=readme-ov-file#hiding-messages-in-the-chat) 部分。

## 添加人工干预

Agent Chat UI 完全支持 [人工干预 （human-in-the-loop）](../concepts/human_in_the_loop.md) 工作流。要进行尝试，请将（[部署教程](../tutorials/langgraph-platform/local-server.md)）`src/agent/graph.py` 文件中的代理代码替换为此 [代理实现](../how-tos/human_in_the_loop/add-human-in-the-loop.md#add-interrupts-to-any-tool)：

<video controls src="../assets/interrupt-chat-ui.mp4" type="video/mp4"></video>

!!! Important

    如果您的 LangGraph 代理使用 @[`HumanInterrupt` schema][HumanInterrupt] 进行中断，Agent Chat UI 的效果最佳。如果您不使用该 schema，Agent Chat UI 将能够渲染传递给 `interrupt` 函数的输入，但它将无法完全支持恢复您的 Graph。

## 生成式 UI

您也可以在 Agent Chat UI 中使用生成式 UI。

生成式 UI 允许您定义 [React](https://react.dev/) 组件，并从 LangGraph 服务器将其推送到 UI。有关构建 LangGraph 生成式 UI 代理的更多文档，请参阅 [这些文档](https://langchain-ai.github.io/langgraph/cloud/how-tos/generative_ui_react/)。