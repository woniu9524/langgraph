---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 运行 Agent

Agent 支持同步和异步执行，可以使用 `.invoke()` / `await .ainvoke()` 获取完整响应，也可以使用 `.stream()` / `.astream()` 进行**增量** [流式](../how-tos/streaming.md)输出。本节将介绍如何提供输入、解释输出、启用流式输出以及控制执行限制。

## 基本用法

Agent 主要有两种执行模式：

:::python

- **同步**：使用 `.invoke()` 或 `.stream()`
- **异步**：使用 `await .ainvoke()` 或 `async for` 配合 `.astream()`
  :::

:::js

- **同步**：使用 `.invoke()` 或 `.stream()`
- **异步**：使用 `await .invoke()` 或 `for await` 配合 `.stream()`
  :::

:::python
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

:::

:::js

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const agent = createReactAgent(...);
// highlight-next-line
const response = await agent.invoke({
    "messages": [
        { "role": "user", "content": "what is the weather in sf" }
    ]
});
```

:::

## 输入和输出

Agent 使用的语言模型期望接收一个 `messages` 列表作为输入。因此，Agent 的输入和输出都存储在 Agent [状态（state）](../concepts/low_level.md#working-with-messages-in-graph-state)的 `messages` 键下，形式为一个消息列表。

## 输入格式

Agent 的输入必须是一个包含 `messages` 键的字典。支持的格式如下：

:::python
| 格式 | 示例 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 字符串 | `{"messages": "Hello"}` — 会被解析为 [HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage) |
| 消息字典 | `{"messages": {"role": "user", "content": "Hello"}}` |
| 消息列表 | `{"messages": [{"role": "user", "content": "Hello"}]}` |
| 配合自定义状态 | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` — 如果使用了自定义 `state_schema` |
:::

:::js
| 格式 | 示例 |
|--------------------|-------------------------------------------------------------------------------------------------------------------------------|
| 字符串 | `{"messages": "Hello"}` — 会被解析为 [HumanMessage](https://js.langchain.com/docs/concepts/messages/#humanmessage) |
| 消息字典 | `{"messages": {"role": "user", "content": "Hello"}}` |
| 消息列表 | `{"messages": [{"role": "user", "content": "Hello"}]}` |
| 配合自定义状态 | `{"messages": [{"role": "user", "content": "Hello"}], "user_name": "Alice"}` — 如果使用了自定义状态定义 |
:::

:::python
消息会自动转换为 LangChain 的内部消息格式。您可以在 LangChain 文档中详细了解 [LangChain 消息](https://python.langchain.com/docs/concepts/messages/#langchain-messages)。
:::

:::js
消息会自动转换为 LangChain 的内部消息格式。您可以在 LangChain 文档中详细了解 [LangChain 消息](https://js.langchain.com/docs/concepts/messages/#langchain-messages)。
:::

!!! tip "使用自定义 Agent 状态"

    :::python
    您可以直接在输入字典中提供 Agent 状态模式（state schema）中定义的额外字段。这可以根据运行时数据或之前的工具输出来实现动态行为。
    有关完整详情，请参阅 [上下文指南](./context.md)。
    :::

    :::js
    您可以直接在状态定义中提供自定义状态中定义的额外字段。这可以根据运行时数据或之前的工具输出来实现动态行为。
    有关完整详情，请参阅 [上下文指南](./context.md)。
    :::

!!! note

    :::python
    传递给 `messages` 的字符串输入会转换为 [HumanMessage](https://python.langchain.com/docs/concepts/messages/#humanmessage)。此行为与 `create_react_agent` 中的 `prompt` 参数不同，后者在传递字符串时会被解析为 [SystemMessage](https://python.langchain.com/docs/concepts/messages/#systemmessage)。
    :::

    :::js
    传递给 `messages` 的字符串输入会转换为 [HumanMessage](https://js.langchain.com/docs/concepts/messages/#humanmessage)。此行为与 `createReactAgent` 中的 `prompt` 参数不同，后者在传递字符串时会被解析为 [SystemMessage](https://js.langchain.com/docs/concepts/messages/#systemmessage)。
    :::

## 输出格式

:::python
Agent 的输出是一个字典，包含：

- `messages`：在执行过程中交换的所有消息列表（用户输入、助手回复、工具调用）。
- 如果配置了 [结构化输出](./agents.md#6-configure-structured-output)，则可选地包含 `structured_response`。
- 如果使用了自定义 `state_schema`，输出中也可能包含对应于您定义字段的其他键。这些键可以存储来自工具执行或提示逻辑的更新状态值。
:::

:::js
Agent 的输出是一个字典，包含：

- `messages`：在执行过程中交换的所有消息列表（用户输入、助手回复、工具调用）。
- 如果配置了 [结构化输出](./agents.md#6-configure-structured-output)，则可选地包含 `structuredResponse`。
- 如果使用了自定义状态定义，输出中也可能包含对应于您定义字段的其他键。这些键可以存储来自工具执行或提示逻辑的更新状态值。
:::

有关处理自定义状态模式及访问上下文的更多详细信息，请参阅 [上下文指南](./context.md)。

## 流式输出

Agent 支持流式响应，以实现更具响应性的应用程序。这包括：

- 每个步骤后的**进度更新**
- 生成过程中的**LLM token**
- 执行期间的**自定义工具消息**

流式输出支持同步和异步模式：

:::python
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

:::

:::js

```typescript
for await (const chunk of agent.stream(
  { messages: [{ role: "user", content: "what is the weather in sf" }] },
  { streamMode: "updates" }
)) {
  console.log(chunk);
}
```

:::

!!! tip

    有关完整详情，请参阅 [流式输出指南](../how-tos/streaming.md)。

## 最大迭代次数

:::python
为了控制 Agent 的执行并避免无限循环，请设置递归限制。这定义了 Agent 在引发 `GraphRecursionError` 之前可以执行的最大步数。您可以在运行时或通过 `.with_config()` 定义 Agent 时配置 `recursion_limit`：
:::

:::js
为了控制 Agent 的執行並避免無限循環，請設置遞歸限制。這定義了 Agent 在引發 `GraphRecursionError` 之前可以執行的最大步數。您可以在運行時或通過 `.withConfig()` 定義 Agent 時配置 `recursionLimit`：
:::

:::python
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

:::

:::js
=== "运行时"

    ```typescript
    import { GraphRecursionError } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/langgraph/prebuilt";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    const maxIterations = 3;
    // highlight-next-line
    const recursionLimit = 2 * maxIterations + 1;
    const agent = createReactAgent({
        llm: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
        tools: [getWeather]
    });

    try {
        const response = await agent.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
            // highlight-next-line
            { recursionLimit }
        );
    } catch (error) {
        if (error instanceof GraphRecursionError) {
            console.log("Agent stopped due to max iterations.");
        }
    }
    ```

=== "`.withConfig()`"

    ```typescript
    import { GraphRecursionError } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/langgraph/prebuilt";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    const maxIterations = 3;
    // highlight-next-line
    const recursionLimit = 2 * maxIterations + 1;
    const agent = createReactAgent({
        llm: new ChatAnthropic({ model: "claude-3-5-haiku-latest" }),
        tools: [getWeather]
    });
    // highlight-next-line
    const agentWithRecursionLimit = agent.withConfig({ recursionLimit });

    try {
        const response = await agentWithRecursionLimit.invoke(
            {"messages": [{"role": "user", "content": "what's the weather in sf"}]},
        );
    } catch (error) {
        if (error instanceof GraphRecursionError) {
            console.log("Agent stopped due to max iterations.");
        }
    }
    ```

:::

:::python

## 更多资源

- [LangChain 中的异步编程](https://python.langchain.com/docs/concepts/async)
  :::