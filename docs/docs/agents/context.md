# 上下文

**上下文工程** 是构建动态系统的实践，这些系统以正确的格式提供正确的信息和工具，以便语言模型能够合理地完成任务。

上下文包括消息列表之外的*任何*数据，这些数据可以影响行为。这些可以是：

- 在运行时传递的信息，例如 `user_id` 或 API 凭据。
- 在多步推理过程中更新的内部状态。
- 先前交互的持久内存或事实。

LangGraph 提供**三种**主要方式来提供上下文：

| 类型                                                                         | 描述                                   | 可变？ | 生命周期            |
|------------------------------------------------------------------------------|----------------------------------------|--------|---------------------|
| [**Config**](#config-static-context)                                         | 在运行开始时传递的数据                 | ❌     | 每个运行          |
| [**短期记忆 (State)**](#short-term-memory-mutable-context)                   | 执行期间可以更改的动态数据             | ✅     | 每个运行或对话    |
| [**长期记忆 (Store)**](#long-term-memory-cross-conversation-context)         | 可以在对话之间共享的数据               | ✅     | 跨对话            |

## 提供运行时上下文

### Config（静态上下文）

Config 用于不可变数据，如用户元数据或 API 密钥。当拥有在运行过程中不会更改的值时使用。

使用一个称为 **"configurable"** 的键指定配置，该键为此目的保留：

```python
graph.invoke( # (1)!
    {"messages": [{"role": "user", "content": "hi!"}]}, # (2)!
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}} # (3)!
)
```

1. 这是代理或图的调用。`invoke` 方法使用提供的输入运行底层图。
2. 此示例使用消息作为输入，这是常见的，但您的应用程序可能使用不同的输入结构。
3. 这是你传递配置数据的地方。`config` 参数允许你提供代理在执行期间可以使用額外外上下文。

=== "Agent prompt"

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    def prompt(state: AgentState, config: RunnableConfig) -> list[AnyMessage]:
        user_name = config["configurable"].get("user_name")
        system_msg = f"You are a helpful assistant. Address the user as {user_name}."
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        prompt=prompt
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        config={"configurable": {"user_name": "John Smith"}}
    )
    ```

    * 有关详细信息，请参阅 [Agents](../agents/agents.md)。

=== "Workflow node"

    ```python
    from langchain_core.runnables import RunnableConfig

    # highlight-next-line
    def node(state: State, config: RunnableConfig):
        user_name = config["configurable"].get("user_name")
        ...
    ```

    * 有关详细信息，请参阅 [Graph API](https://langchain-ai.github.io/langgraph/how-tos/graph-api/#add-runtime-configuration)。

=== "In a tool"

    ```python
    from langchain_core.runnables import RunnableConfig

    @tool
    # highlight-next-line
    def get_user_info(config: RunnableConfig) -> str:
        """Retrieve user information based on user ID."""
        user_id = config["configurable"].get("user_id")
        return "User is John Smith" if user_id == "user_123" else "Unknown user"
    ```

    有关详细信息，请参阅 [tool calling guide](../how-tos/tool-calling.md#configuration)。

### 短期记忆（可变上下文）

State 在运行期间充当[短期记忆](../concepts/memory.md)。它包含在执行过程中可能演变的数据，例如来自工具或 LLM 输出的值。

=== "In an agent"

    示例展示如何将状态合并到代理**prompt**中。

    代理的**tools**也可以访问状态，工具可以根据需要读取或更新状态。有关详细信息，请参阅 [tool calling guide](../how-tos/tool-calling.md#short-term-memory)。

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

    1. 定义一个扩展 `AgentState` 或 `MessagesState` 的自定义状态。
    2. 将自定义状态模式传递给代理。这允许代理在执行期间访问和修改状态。


=== "In a workflow"

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
    
    1. 定义一个自定义状态
    2. 在任何节点或工具中访问状态
    3. Graph API 的设计宗旨是尽可能轻松地与状态协同工作。节点返回值代表对状态的请求更新。


!!! tip "启用记忆"

    有关如何启用内存的更多详细信息，请参阅[内存指南](../how-tos/memory/add-memory.md)。这是一个强大的功能，它允许你在多次调用之间保持代理的状态。否则，状态仅限于单个运行。

### 长期记忆（跨对话上下文）

对于跨越对话或会话的上下文，LangGraph 允许通过 `store` 访问**长期记忆**。这可用于读取或更新持久事实（例如 用户配置文件、偏好设置、先前的交互）。

有关更多信息，请参阅[内存指南](../how-tos/memory/add-memory.md)。