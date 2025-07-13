---
search:
  boost: 2
---

# Agent 架构

许多 LLM 应用在调用 LLM 之前和/或之后实现特定的步骤控制流。例如，[RAG](https://github.com/langchain-ai/rag-from-scratch) 会检索与用户问题相关的文档，并将这些文档传递给 LLM，以便将模型的响应锚定在提供的文档上下文中。

我们不希望硬编码固定的控制流，而是希望 LLM 系统能够自行选择控制流来解决更复杂的问题！这是 [agent](https://blog.langchain.dev/what-is-an-agent/) 的一个定义：*agent 是一个使用 LLM 来决定应用程序控制流的系统。* LLM 有许多控制应用程序的方式：

- LLM 可以在两条潜在路径之间进行路由
- LLM 可以决定调用哪个工具
- LLM 可以决定生成的答案是否足够，或者是否需要更多工作

因此，存在许多不同类型的 [agent 架构](https://blog.langchain.dev/what-is-a-cognitive-architecture/)，它们赋予 LLM 不同程度的控制。

![Agent Types](img/agent_types.png)

## Router

Router 允许 LLM 从一组指定的选项中选择一个单一的步骤。这是一种控制程度相对有限的 agent 架构，因为 LLM 通常专注于做出单个决策，并从一组预定义的有限选项中生成特定的输出。Router 通常采用几种不同的概念来实现这一点。

### Structured Output

带有 LLM 的结构化输出通过提供 LLM 应遵循的特定格式或模式来工作。这类似于工具调用，但更通用。虽然工具调用通常涉及选择和使用预定义的函数，但结构化输出可用于任何类型的格式化响应。实现结构化输出的常用方法包括：

1. Prompt engineering：通过系统提示指示 LLM 以特定格式响应。
2. Output parsers：使用后处理从 LLM 响应中提取结构化数据。
3. Tool calling：利用某些 LLM 内置的工具调用功能生成结构化输出。

结构化输出对于路由至关重要，因为它们确保 LLM 的决策能够被系统可靠地解释和执行。通过此操作指南了解有关 [structured outputs](https://python.langchain.com/docs/how_to/structured_output/) 的更多信息。

## Tool-calling Agent

虽然 Router 允许 LLM 做出单个决策，但更复杂的 Agent 架构通过两种关键方式扩展了 LLM 的控制：

1. Multi-step decision making：LLM 可以连续做出多个决策，而不仅仅是一个。
2. Tool access：LLM 可以选择和使用各种工具来完成任务。

[ReAct](https://arxiv.org/abs/2210.03629) 是一种流行的通用 Agent 架构，它结合了这些扩展，集成了三个核心概念。

1. [Tool calling](#tool-calling)：允许 LLM 根据需要选择和使用各种工具。
2. [Memory](#memory)：使 Agent 能够保留和使用先前步骤中的信息。
3. [Planning](#planning)：使 LLM 能够创建和遵循多步计划以实现目标。

这种架构允许更复杂和灵活的 Agent 行为，超越了简单的路由，能够通过多个步骤实现动态问题解决。与最初的 [论文](https://arxiv.org/abs/2210.03629) 不同，今天的 Agent 依赖 LLM 的 [tool calling](#tool-calling) 能力，并在 [messages](./low_level.md#why-use-messages) 列表上运行。

在 LangGraph 中，您可以使用预构建的 [agent](../agents/agents.md#2-create-an-agent) 来开始使用 tool-calling agent。

### Tool calling

当您希望 Agent 与外部系统交互时，工具会非常有用。外部系统（例如 API）通常需要特定的输入模式或负载，而不是自然语言。例如，当我们把一个 API 绑定为一个工具时，我们就让模型了解所需的输入模式。模型将根据用户的自然语言输入选择调用工具，并返回符合工具所需模式的输出。

[许多 LLM 提供商都支持工具调用](https://python.langchain.com/docs/integrations/chat/)，并且 LangChain 中的 [工具调用接口](https://blog.langchain.dev/improving-core-tool-interfaces-and-docs-in-langchain/) 非常简单：您只需将任何 Python `function` 传递给 `ChatModel.bind_tools(function)`。

![Tools](img/tool_call.png)

### Memory

[Memory](../how-tos/memory/add-memory.md) 对于 Agent 至关重要，它使 Agent 能够在问题解决的多个步骤中保留和利用信息。它在不同的尺度上运行：

1. [Short-term memory](../how-tos/memory/add-memory.md#add-short-term-memory)：允许 Agent 访问在序列中早期步骤中获取的信息。
2. [Long-term memory](../how-tos/memory/add-memory.md#add-long-term-memory)：使 Agent 能够回忆起先前交互中的信息，例如对话中的过去消息。

LangGraph 对内存实现提供了完全控制：

- [`State`](./low_level.md#state)：用户定义的模式，指定要保留的内存的确切结构。
- [`Checkpointer`](./persistence.md#checkpoints)：一种在会话的交互过程中，在每个步骤存储状态的机制。
- [`Store`](./persistence.md#memory-store)：一种在会话之间存储用户特定或应用程序级别数据的机制。

这种灵活的方法允许您根据特定 Agent 架构的需求定制内存系统。有效的内存管理可以增强 Agent 保持上下文、从过去经验中学习以及随着时间推移做出更明智决策的能力。有关添加和管理内存的实用指南，请参阅 [Memory](../how-tos/memory/add-memory.md)。

### Planning

在 tool-calling [agent](../agents/overview.md#what-is-an-agent) 中，LLM 会在一个 while 循环中被反复调用。在每个步骤中，Agent 会决定调用哪些工具以及这些工具的输入是什么。然后执行这些工具，并将输出作为观察结果反馈给 LLM。当 Agent 决定它已获得足够信息来解决用户请求，并且不值得再调用任何工具时，while 循环就会终止。

## Custom Agent Architectures

虽然 Router 和 Tool-calling Agent（如 ReAct）很常见，但 [定制 Agent 架构](https://blog.langchain.dev/why-you-should-outsource-your-agentic-infrastructure-but-own-your-cognitive-architecture/) 通常能为特定任务带来更好的性能。LangGraph 提供了多种强大功能来构建定制的 Agent 系统：

### Human-in-the-loop

人为干预可以显著提高 Agent 的可靠性，尤其是在处理敏感任务时。这可能包括：

- 批准特定操作
- 向 Agent 的状态提供反馈以进行更新
- 在复杂的决策过程中提供指导

当完全自动化不可行或不可取时，人为干预模式至关重要。在该 [human-in-the-loop guide](./human_in_the_loop.md) 中了解更多信息。

### Parallelization

并行处理对于高效的多 Agent 系统和复杂任务至关重要。LangGraph 通过其 [Send](./low_level.md#send) API 支持并行处理，从而实现：

- 并发处理多个状态
- 实现类似 map-reduce 的操作
- 高效处理独立的子任务

有关实际实现，请参阅我们的 [map-reduce tutorial](../how-tos/graph-api.md#map-reduce-and-the-send-api)

### Subgraphs

[Subgraphs](./subgraphs.md) 对于管理复杂的 Agent 架构至关重要，尤其是在 [multi-agent systems](./multi_agent.md) 中。它们允许：

- 为单个 Agent 进行隔离的状态管理
- Agent 团队的层次化组织
- Agent 和主系统之间的受控通信

Subgraphs 通过状态模式中的重叠键与父图进行通信。这使得 Agent 设计具有灵活性和模块化。有关实现细节，请参阅我们的 [subgraph how-to guide](../how-tos/subgraph.md)。

### Reflection

反射机制可以通过以下方式显著提高 Agent 的可靠性：

1. 评估任务完成情况和正确性
2. 为迭代改进提供反馈
3. 实现自我纠正和学习

虽然通常基于 LLM，但反射也可以使用确定性方法。例如，在编码任务中，编译错误可以作为反馈。这种方法在 [this video using LangGraph for self-corrective code generation](https://www.youtube.com/watch?v=MvNdgmM7uyc) 中得到了演示。

通过利用这些功能，LangGraph 可以创建复杂的、任务特定的 Agent 架构，这些架构能够处理复杂的工作流、有效地协作并持续改进其性能。