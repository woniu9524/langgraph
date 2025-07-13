---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# LangGraph 快速入门

本指南将向您展示如何设置和使用 LangGraph 的**预构建**、**可复用**组件，这些组件旨在帮助您快速可靠地构建 Agentic 系统。

## 前提条件

在开始本教程之前，请确保您已具备以下条件：

- Anthropic API 密钥

## 1. 安装依赖

如果您尚未安装 LangGraph 和 LangChain，请安装它们：

```
pip install -U langgraph "langchain[anthropic]"
```

!!! info 

    安装 LangChain 是为了让 Agent 可以调用 [模型](https://python.langchain.com/docs/integrations/chat/)。

## 2. 创建 Agent

要创建 Agent，请使用 `create_react_agent`：

```python
from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:  # (1)!
    """获取给定城市的]天气。"""
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",  # (2)!
    tools=[get_weather],  # (3)!
    prompt="你是一个乐于助人的助手"  # (4)!
)

# 运行 Agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

1. 定义 Agent 可以使用的工具。工具可以定义为普通的 Python 函数。有关更高级的工具使用和自定义方法，请查看 [工具](../how-tos/tool-calling.md) 页面。
2. 提供 Agent 使用的语言模型。要了解有关为 Agent 配置语言模型的更多信息，请查看 [模型](./models.md) 页面。
3. 提供模型要使用的工具列表。
4. 为 Agent 使用的语言模型提供一个系统提示（说明）。

## 3. 配置 LLM

要使用特定参数配置 LLM，例如 temperature，请使用 `init_chat_model`：

```python
from langchain.chat_models import init_chat_model
from langgraph.prebuilt import create_react_agent

# highlight-next-line
model = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    # highlight-next-line
    temperature=0
)

agent = create_react_agent(
    # highlight-next-line
    model=model,
    tools=[get_weather],
)
```

有关如何配置 LLM 的更多信息，请参阅 [模型](./models.md)。

## 4. 添加自定义提示

提示用于指示 LLM 如何表现。可以添加以下任一类型的提示：

*   **静态提示**：字符串被解释为**系统消息**。
*   **动态提示**：基于输入或配置在**运行时**生成的消息列表。

=== "静态提示"

    定义一个固定的提示字符串或消息列表：

    ```python
    from langgraph.prebuilt import create_react_agent

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        # 一个永远不会改变的静态提示
        # highlight-next-line
        prompt="永远不要回答关于天气的问题。"
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
    )
    ```

=== "动态提示"

    定义一个函数，该函数根据 Agent 的状态和配置返回消息列表：

    ```python
    from langchain_core.messages import AnyMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    def prompt(state: AgentState, config: RunnableConfig) -> list[AnyMessage]:  # (1)!
        user_name = config["configurable"].get("user_name")
        system_msg = f"你是一个乐于助人的助手。请称呼用户为 {user_name}。"
        return [{"role": "system", "content": system_msg}] + state["messages"]

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_weather],
        # highlight-next-line
        prompt=prompt
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
        # highlight-next-line
        config={"configurable": {"user_name": "John Smith"}}
    )
    ```

    1. 动态提示允许在构建 LLM 输入时包含非消息的[上下文](./context.md)，例如：

        - 在运行时传递的信息，如 `user_id` 或 API 凭证（使用 `config`）。
        - 在多步推理过程中更新的内部 Agent 状态（使用 `state`）。

        动态提示可以定义为接收 `state` 和 `config` 并返回要发送到 LLM 的消息列表的函数。

有关更多信息，请参阅 [上下文](./context.md)。

## 5. 添加内存

要允许与 Agent 进行多轮对话，您需要通过在创建 Agent 时提供 `checkpointer` 来启用[持久化](../concepts/persistence.md)。在运行时，您需要提供一个包含 `thread_id` 的配置——这是对话（会话）的唯一标识符：

```python
from langgraph.prebuilt import create_react_agent
from langgraph.checkpoint.memory import InMemorySaver

# highlight-next-line
checkpointer = InMemorySaver()

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    # highlight-next-line
    checkpointer=checkpointer  # (1)!
)

# 运行 Agent
# highlight-next-line
config = {"configurable": {"thread_id": "1"}}
sf_response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
    # highlight-next-line
    config  # (2)!
)
ny_response = agent.invoke(
    {"messages": [{"role": "user", "content": "what about new york?"}]},
    # highlight-next-line
    config
)
```

1. `checkpointer` 允许 Agent 在工具调用循环的每个步骤中存储其状态。这支持[短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory)和[人工干预](../concepts/human_in_the_loop.md)功能。
2. 传递带有 `thread_id` 的配置，以便在 Agent 的后续调用中可以继续相同的对话。

启用 checkpointer 后，它会在提供的 checkpointer 数据库（如果在内存中使用 `InMemorySaver` 则在内存中）存储每个步骤的 Agent 状态。

请注意，在上面的示例中，当 Agent 在第二次使用相同的 `thread_id` 调用时，第一次对话的原始消息历史记录以及新的用户输入会自动包含在内。

有关更多信息，请参阅 [内存](../how-tos/memory/add-memory.md)。

## 6. 配置结构化输出

要生成符合模式的结构化响应，请使用 `response_format` 参数。可以使用 `Pydantic` 模型或 `TypedDict` 定义模式。结果可以通过 `structured_response` 字段访问。

```python
from pydantic import BaseModel
from langgraph.prebuilt import create_react_agent

class WeatherResponse(BaseModel):
    conditions: str

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    # highlight-next-line
    response_format=WeatherResponse  # (1)!
)

response = agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)

# highlight-next-line
response["structured_response"]
```

1. 当提供 `response_format` 时，会在 Agent 循环末尾添加一个额外的步骤：Agent 消息历史记录会与结构化输出一起传递给 LLM，以生成结构化响应。

    要为此 LLM 提供系统提示，请使用元组 `(prompt, schema)`，例如 `response_format=(prompt, WeatherResponse)`。

!!! Note "LLM 后处理"

    结构化输出需要额外调用 LLM 来根据模式格式化响应。

## 后续步骤

- [在本地部署您的 Agent](../tutorials/langgraph-platform/local-server.md)
- [了解更多关于预构建 Agent 的信息](../agents/overview.md)
- [LangGraph Platform 快速入门](../cloud/quick_start.md)