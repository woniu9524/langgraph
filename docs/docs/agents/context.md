# 上下文

**上下文工程**（Context engineering）是指构建动态系统，以正确的格式提供正确的信息和工具，从而使 AI 应用程序能够完成任务。上下文可以沿两个关键维度进行表征：

1.  按 **可变性**（mutability）：
    *   **静态上下文**（Static context）：在执行期间不发生变化的不可变数据（例如，用户元数据、数据库连接、工具）。
    *   **动态上下文**（Dynamic context）：随着应用程序运行而演变的变动数据（例如，对话历史、中间结果、工具调用观察）。
2.  按 **生命周期**（lifetime）：
    *   **运行时上下文**（Runtime context）：作用于单个运行或调用的数据。
    *   **跨会话上下文**（Cross-conversation context）：在多个会话或期间持久存在的数据。

!!! tip "运行时上下文 vs LLM 上下文"

    运行时上下文指的是本地上下文：代码运行所需的 **数据和依赖项**。它 **不** 指的是：

    *   **LLM 上下文**：传递给 LLM 提示的数据。
    *   **"上下文窗口"**：可以传递给 LLM 的最大 token 数量。

    运行时上下文可用于优化 LLM 上下文。例如，您可以在运行时上下文中利用用户元数据来获取用户偏好，并将其输入到上下文窗口中。

LangGraph 提供了三种管理上下文的方法，这些方法结合了可变性和生命周期维度：

:::python

| 上下文类型                                                                          | 描述                                                                  | 可变性 | 生命周期           | 访问方式                                  |
| :---------------------------------------------------------------------------------- | :-------------------------------------------------------------------- | :----- | :----------------- | :---------------------------------------- |
| [**静态运行时上下文**](#static-runtime-context)                                     | 在启动时传入的用户元数据、工具、数据库连接                            | 静态   | 单次运行         | `invoke`/`stream` 的 `context` 参数             |
| [**动态运行时上下文（状态）**](#dynamic-runtime-context-state)                        | 在单次运行中演变的变动数据                                            | 动态   | 单次运行         | LangGraph 状态对象                          |
| [**动态跨会话上下文（存储）**](#dynamic-cross-conversation-context-store)             | 在多个会话之间持久存在的数据                                          | 动态   | 跨会话           | LangGraph 存储 (store)                      |

## 静态运行时上下文

**静态运行时上下文** 代表不可变的数据，如用户元数据、工具和数据库连接，这些数据在运行开始时通过 `invoke`/`stream` 的 `context` 参数传递给应用程序。这些数据在执行过程中不会改变。

!!! version-added "0.6.0 版本添加：`context` 替换 `config['configurable']`"

    现在运行时上下文通过 `invoke`/`stream` 的 `context` 参数传递，该参数取代了之前将应用程序配置传递给 `config['configurable']` 的模式。

```python
@dataclass
class ContextSchema:
    user_name: str

graph.invoke( # (1)!
    {"messages": [{"role": "user", "content": "hi!"}]}, # (2)!
    # highlight-next-line
    context={"user_name": "John Smith"} # (3)!
)
```

1.  这是代理或图的调用。`invoke` 方法使用提供的输入运行底层图。
2.  此示例使用消息作为输入，这很常见，但您的应用程序可能使用不同的输入结构.
3.  这是您传递运行时数据的地方。`context` 参数允许您提供代理在其执行过程中可以使用的其他依赖项。

=== "Agent prompt"

    ```python
    from langchain_core.messages import AnyMessage
    from langgraph.runtime import get_runtime
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    def prompt(state: AgentState) -> list[AnyMessage]:
        runtime = get_runtime(ContextSchema)
        system_msg = f"You are a helpful assistant. Address the user as {runtime.context.user_name}."
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        prompt=prompt,
        context_schema=ContextSchema
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        context={"user_name": "John Smith"}
    )
    ```

    *   有关详细信息，请参阅[代理 (Agents)](../agents/agents.md)。

=== "Workflow node"

    ```python
    from langgraph.runtime import Runtime

    # highlight-next-line
    def node(state: State, runtime: Runtime[ContextSchema]):
        user_name = runtime.context.user_name
        ...
    ```

    *   有关详细信息，请参阅[图 API (Graph API)](https://langchain-ai.github.io/langgraph/how-tos/graph-api/#add-runtime-configuration)。

=== "In a tool"

    ```python
    from langgraph.runtime import get_runtime

    @tool
    # highlight-next-line
    def get_user_email() -> str:
        """Retrieve user information based on user ID."""
        # simulate fetching user info from a database
        runtime = get_runtime(ContextSchema)
        email = get_user_email_from_db(runtime.context.user_name)
        return email
    ```

    有关详细信息，请参阅[工具调用指南 (tool calling guide](../how-tos/tool-calling.md#configuration)。

!!! tip

    `Runtime` 对象可用于访问静态上下文和其他实用程序，如活动存储和流写入器。
    有关详细信息，请参阅[Runtime][langgraph.runtime.Runtime] 文档。

:::

:::js

| 上下文类型                                                                          | 描述                       | 可变性 | 生命周期           |
| :---------------------------------------------------------------------------------- | :------------------------- | :----- | :----------------- |
| [**配置 (Config)**](#config-static-context)                                         | 运行开始时传入的数据       | 静态   | 单次运行         |
| [**动态运行时上下文（状态）**](#dynamic-runtime-context-state)                        | 在单次运行中演变的变动数据 | 动态   | 单次运行         |
| [**动态跨会话上下文（存储）**](#dynamic-cross-conversation-context-store)             | 在多个会话之间持久存在的数据 | 动态   | 跨会话           |

## 配置 (Config) (静态上下文)

Config 用于不可变数据，如用户元数据或 API 密钥。当您拥有在运行过程中不改变的值时，请使用此选项。

使用一个名为 **"configurable"** 的保留键来指定配置。

```typescript
await graph.invoke(
  // (1)!
  { messages: [{ role: "user", content: "hi!" }] }, // (2)!
  // highlight-next-line
  { configurable: { user_id: "user_123" } } // (3)!
);
```

:::

## 动态运行时上下文（状态）

**动态运行时上下文** 代表在单次运行中可以演变的变动数据，并通过 LangGraph 状态对象进行管理。这包括对话历史、中间结果以及从工具或 LLM 输出派生的值。在 LangGraph 中，状态对象充当运行期间的[短期记忆](../concepts/memory.md)。

=== "在代理中"

    示例显示如何将状态整合到代理的**提示**中。

    代理的**工具**也可以访问状态，这些工具可以根据需要读取或更新状态。有关详细信息，请参阅[工具调用指南 (tool calling guide](../how-tos/tool-calling.md#short-term-memory)。

    :::python
    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt import create_react_agent
    from langgraph.prebuilt.chat_agent_executor import AgentState

    # highlight-next-line
    class CustomState(AgentState): # (1)!
        user_name: str

    def prompt(
        # highlight-next-line
        state: CustomState
    ) -> list[AnyMessage]:
        user_name = state["user_name"]
        system_msg = f"You are a helpful assistant. User's name is {user_name}"
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[...],
        # highlight-next-line
        state_schema=CustomState, # (2)!
        prompt=prompt
    )

    agent.invoke({
        "messages": "hi!",
        "user_name": "John Smith"
    })
    ```

    1.  定义一个扩展 `AgentState` 或 `MessagesState` 的自定义状态模式。
    2.  将自定义状态模式传递给代理。这允许代理在执行期间访问和修改状态。
    :::

    :::js
    ```typescript
    import type { BaseMessage } from "@langchain/core/messages";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { MessagesZodState } from "@langchain/langgraph";
    import { z } from "zod";

    // highlight-next-line
    const CustomState = z.object({ // (1)!
      messages: MessagesZodState.shape.messages,
      userName: z.string(),
    });

    const prompt = (
      // highlight-next-line
      state: z.infer<typeof CustomState>
    ): BaseMessage[] => {
      const userName = state.userName;
      const systemMsg = `You are a helpful assistant. User's name is ${userName}`;
      return [{ role: "system", content: systemMsg }, ...state.messages];
    };

    const agent = createReactAgent({
      llm: model,
      tools: [...],
      // highlight-next-line
      stateSchema: CustomState, // (2)!
      stateModifier: prompt,
    });

    await agent.invoke({
      messages: [{ role: "user", content: "hi!" }],
      userName: "John Smith",
    });
    ```

    1.  定义一个扩展 `MessagesZodState` 或创建新模式的自定义状态模式。
    2.  将自定义状态模式传递给代理。这允许代理在执行期间访问和修改状态。
    :::

=== "在工作流中"

    :::python
    ```python
    from typing_extensions import TypedDict
    from langchain_core.messages import AnyMessage
    from langgraph.graph import StateGraph

    # highlight-next-line
    class CustomState(TypedDict): # (1)!
        messages: list[AnyMessage]
        extra_field: int

    # highlight-next-line
    def node(state: CustomState): # (2)!
        messages = state["messages"]
        ...
        return { # (3)!
            # highlight-next-line
            "extra_field": state["extra_field"] + 1
        }

    builder = StateGraph(State)
    builder.add_node(node)
    builder.set_entry_point("node")
    graph = builder.compile()
    ```

    1.  定义一个自定义状态。
    2.  在任何节点或工具中访问状态。
    3.  图 API (Graph API) 的设计宗旨是尽可能轻松地使用状态。节点返回的值代表对状态的请求更新。
    :::

    :::js
    ```typescript
    import type { BaseMessage } from "@langchain/core/messages";
    import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
    import { z } from "zod";

    // highlight-next-line
    const CustomState = z.object({ // (1)!
      messages: MessagesZodState.shape.messages,
      extraField: z.number(),
    });

    const builder = new StateGraph(CustomState)
      .addNode("node", async (state) => { // (2)!
        const messages = state.messages;
        // ...
        return { // (3)!
          // highlight-next-line
          extraField: state.extraField + 1,
        };
      })
      .addEdge(START, "node");

    const graph = builder.compile();
    ```

    1.  定义一个自定义状态。
    2.  在任何节点或工具中访问状态。
    3.  图 API (Graph API) 的设计宗旨是尽可能轻松地使用状态。节点返回的值代表对状态的请求更新。
    :::

!!! tip "开启记忆"

    有关更多详细信息，请参阅[记忆指南 (memory guide](../how-tos/memory/add-memory.md)。这是一个强大的功能，允许您跨多个调用持久化代理的状态。否则，状态仅限于单个运行。

## 动态跨会话上下文（存储）

**动态跨会话上下文** 代表跨多个会话或期间持久存在的、可变的（mutable）数据，并通过 LangGraph 存储进行管理。这包括用户配置文件、偏好设置和历史交互。LangGraph 存储充当多个运行中的[长期记忆](../concepts/memory.md#long-term-memory)。这可用于读取或更新持久的事实（例如，用户配置文件、偏好设置、先前的交互）。

有关更多信息，请参阅[记忆指南 (Memory guide](../how-tos/memory/add-memory.md)。