---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 运行 Agent


Agent 支持使用 `.invoke()` / `await .ainvoke()` 同步或异步执行以获取完整响应，或使用 `.stream()` / `.astream()` 获取**增量** [流式](../how-tos/streaming.md)输出。本节将介绍如何提供输入、解析输出、启用流式输出以及控制执行限制。


## 基本用法

Agent 可以通过两种主要模式执行：

- **同步** 使用 `.invoke()` 或 `.stream()`
- **异步** 使用 `await .ainvoke()` 或 `async for` 与 `.astream()`

=== "同步调用"
    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)

    # highlight-next-line
    response = agent.invoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

=== "异步调用"
    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(...)
    # highlight-next-line
    response = await agent.ainvoke({"messages": [{"role": "user", "content": "what is the weather in sf"}]})
    ```

## 输入和输出

Agent 使用需要 `messages` 列表作为输入的语言模型。因此，Agent 的输入和输出存储在 Agent [状态](../concepts/low_level.md#working-with-messages-in-graph-state)的 `messages` 键下的消息列表中。

## 输入格式

Agent 输入必须是一个带有 `messages` 键的字典。支持的格式如下：

| 格式             | 示例                                                                                                                       |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 字符串             | `{"messages": "Hello"}`  — 被解释为 [HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage) |
| 消息字典           | `{"messages": {"role": "user", "content": "Hello"}}`                                                                          |
| 消息列表           | `{"messages": [{"role": "user", "content": "Hello"}]}`                                                                        |
| 自定义状态         | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` — 如果使用自定义 `state_schema`               |

消息会自动转换为 LangChain 的内部消息格式。您可以在 LangChain 文档中详细了解 [LangChain 消息](https://python.langchain.com/docs/concepts/messages/#langchain-messages)。

!!! tip "使用自定义 Agent 状态"

    您可以直接在输入字典中提供 Agent 状态模式中定义的附加字段。这允许基于运行时数据或先前工具输出来实现动态行为。
    有关完整详细信息，请参阅 [上下文指南](./context.md)。

!!! note

    `messages` 的字符串输入会被转换为 [HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage)。此行为与 `create_react_agent` 中的 `prompt` 参数不同，后者在作为字符串传递时被解释为 [SystemMessage](https://python.langchain.com/docs/concepts/messages/#systemmessage)。

## 输出格式

Agent 输出是一个包含以下内容的字典：

- `messages`: 执行期间交换的所有消息列表（用户输入、助手回复、工具调用）。
- 可选地，`structured_response`，如果配置了 [结构化输出](./agents.md#6-configure-structured-output)。
- 如果使用自定义 `state_schema`，与您定义的字段对应的附加键也可能出现在输出中。这些可以包含来自工具执行或提示逻辑的更新状态值。

有关使用自定义状态模式和访问上下文的更多详细信息，请参阅 [上下文指南](./context.md)。

## 流式输出

Agent 支持流式响应，以实现更具响应性的应用程序。这包括：

- 每个步骤后的**进度更新**
- 生成时的**LLM 令牌**
- 执行期间的**自定义工具消息**

流式输出在同步和异步模式下均可用：

=== "同步流式输出"

    ```python
    for chunk in agent.stream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

=== "异步流式输出"

    ```python
    async for chunk in agent.astream(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        stream_mode="updates"
    ):
        print(chunk)
    ```

!!! tip

    有关完整详细信息，请参阅 [流式输出指南](../how-tos/streaming.md)。

## 最大迭代次数

为了控制 Agent 的执行并避免无限循环，请设置递归限制。这定义了 Agent 在引发 `GraphRecursionError` 之前可以执行的最大步骤数。您可以在运行时或通过 `.with_config()` 定义 Agent 时配置 `recursion_limit`：

=== "运行时"

    ```python
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    # highlight-next-line
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )

    try:
        response = agent.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
            # highlight-next-line
            {"recursion_limit": recursion_limit},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

=== "`.with_config()`"

    ```python
    from langgraph.errors import GraphRecursionError
    from langgraph.prebuilt import create_react_agent

    max_iterations = 3
    # highlight-next-line
    recursion_limit = 2 * max_iterations + 1
    agent = create_react_agent(
        model="anthropic:claude-3-5-haiku-latest",
        tools=[get_weather]
    )
    # highlight-next-line
    agent_with_recursion_limit = agent.with_config(recursion_limit=recursion_limit)

    try:
        response = agent_with_recursion_limit.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
        )
    except GraphRecursionError:
        print("Agent stopped due to max iterations.")
    ```

## 附加资源

* [LangChain 中的异步编程](https://python.langchain.com/docs/concepts/async)