# 调用工具

[工具](../concepts/tools.md)封装了可调用的函数及其输入模式。这些工具可以传递给兼容的 [聊天模型](https://python.langchain.com/docs/concepts/chat_models)，使模型能够决定是否调用某个工具并确定适当的参数。

您可以 [定义自己的工具](#define-a-tool) 或使用 [预建工具](#prebuilt-tools)。

## 定义工具

使用 [@tool](https://python.langchain.com/api_reference/core/tools/langchain_core.tools.convert.tool.html) 装饰器定义一个基本工具：

```python
from langchain_core.tools import tool

# highlight-next-line
@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

## 运行工具

工具符合 [Runnable 接口](https://python.langchain.com/docs/concepts/runnables/)，这意味着您可以使用 `invoke` 方法运行工具：

```python
multiply.invoke({"a": 6, "b": 7})  # 返回 42
```

如果使用 `type="tool_call"` 调用工具，它将返回一个 [ToolMessage](https://python.langchain.com/docs/concepts/messages/#toolmessage)：

```python
tool_call = {
    "type": "tool_call",
    "id": "1",
    "args": {"a": 42, "b": 7}
}
multiply.invoke(tool_call) # 返回一个 ToolMessage 对象
```

输出：

```pycon
ToolMessage(content='294', name='multiply', tool_call_id='1')
```

## 在代理中使用

要创建调用工具的代理，您可以使用预建的 [create_react_agent][langgraph.prebuilt.chat_agent_executor.create_react_agent]：

```python
from langchain_core.tools import tool
# highlight-next-line
from langgraph.prebuilt import create_react_agent

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

# highlight-next-line
agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet",
    tools=[multiply]
)
agent.invoke({"messages": [{"role": "user", "content": "what's 42 x 7?"}]})
```

## 在工作流中使用

如果您正在编写自定义工作流，则需要：

1. 将工具注册到聊天模型
2. 如果模型决定使用该工具，则调用它

使用 `model.bind_tools()` 将工具注册到模型。

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(model="claude-3-5-haiku-latest")

# highlight-next-line
model_with_tools = model.bind_tools([multiply])
```

LLM 会自动确定是否需要调用工具，并处理使用适当参数调用该工具。

??? example "扩展示例：将工具附加到聊天模型"

    ```python
    from langchain_core.tools import tool
    from langchain.chat_models import init_chat_model

    @tool
    def multiply(a: int, b: int) -> int:
        """Multiply two numbers."""
        return a * b

    model = init_chat_model(model="claude-3-5-haiku-latest")
    # highlight-next-line
    model_with_tools = model.bind_tools([multiply])

    response_message = model_with_tools.invoke("what's 42 x 7?")
    tool_call = response_message.tool_calls[0]

    multiply.invoke(tool_call)
    ```

    ```pycon
    ToolMessage(
        content='294',
        name='multiply',
        tool_call_id='toolu_0176DV4YKSD8FndkeuuLj36c'
    )
    ```
#### ToolNode

要在自定义工作流中执行工具，请使用预建的 [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode] 或实现自己的自定义节点。

`ToolNode` 是一个用于在工作流中执行工具的专用节点。它提供以下功能：

* 支持同步和异步工具。
* 并行执行多个工具。
* 处理工具执行期间的错误 (`handle_tool_errors=True`，默认启用)。有关更多详细信息，请参阅 [处理工具错误](#handle-errors)。

`ToolNode` 操作于 [`MessagesState`](../concepts/low_level.md#messagesstate)：

* **输入**：`MessagesState`，其中最后一个消息是包含 `tool_calls` 参数的 `AIMessage`。
* **输出**：`MessagesState`，并使用从已执行工具返回的 [`ToolMessage`](https://python.langchain.com/docs/concepts/messages/#toolmessage) 进行更新。

```python
# highlight-next-line
from langgraph.prebuilt import ToolNode

def get_weather(location: str):
    """调用以获取当前天气。"""
    if location.lower() in ["sf", "san francisco"]:
        return "It's 60 degrees and foggy."
    else:
        return "It's 90 degrees and sunny."

def get_coolest_cities():
    """获取最酷城市的列表"""
    return "nyc, sf"

# highlight-next-line
tool_node = ToolNode([get_weather, get_coolest_cities])
tool_node.invoke({"messages": [...]})
```

??? example "单个工具调用"

    ```python
    from langchain_core.messages import AIMessage
    from langgraph.prebuilt import ToolNode
    
    # 定义工具
    @tool
    def get_weather(location: str):
        """调用以获取当前天气。"""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."
    
    # highlight-next-line
    tool_node = ToolNode([get_weather])
    
    message_with_single_tool_call = AIMessage(
        content="",
        tool_calls=[
            {
                "name": "get_weather",
                "args": {"location": "sf"},
                "id": "tool_call_id",
                "type": "tool_call",
            }
        ],
    )
    
    tool_node.invoke({"messages": [message_with_single_tool_call]})
    ```
    
    ```
    {'messages': [ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id')]}
    ```

??? example "多个工具调用"

    ```python
    from langchain_core.messages import AIMessage
    from langgraph.prebuilt import ToolNode
    
    # 定义工具
    
    def get_weather(location: str):
        """调用以获取当前天气。"""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."
    
    def get_coolest_cities():
        """获取最酷城市的列表"""
        return "nyc, sf"
    
    # highlight-next-line
    tool_node = ToolNode([get_weather, get_coolest_cities])

    message_with_multiple_tool_calls = AIMessage(
        content="",
        tool_calls=[
            {
                "name": "get_coolest_cities",
                "args": {},
                "id": "tool_call_id_1",
                "type": "tool_call",
            },
            {
                "name": "get_weather",
                "args": {"location": "sf"},
                "id": "tool_call_id_2",
                "type": "tool_call",
            },
        ],
    )

    # highlight-next-line
    tool_node.invoke({"messages": [message_with_multiple_tool_calls]})  # (1)!
    ```

    1. `ToolNode` 将并行执行这两个工具。

    ```
    {
        'messages': [
            ToolMessage(content='nyc, sf', name='get_coolest_cities', tool_call_id='tool_call_id_1'),
            ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id_2')
        ]
    }
    ```

??? example "与聊天模型一起使用"

    ```python
    from langchain.chat_models import init_chat_model
    from langgraph.prebuilt import ToolNode
    
    def get_weather(location: str):
        """调用以获取当前天气。"""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."
    
    # highlight-next-line
    tool_node = ToolNode([get_weather])
    
    model = init_chat_model(model="claude-3-5-haiku-latest")
    # highlight-next-line
    model_with_tools = model.bind_tools([get_weather])  # (1)!
    
    
    # highlight-next-line
    response_message = model_with_tools.invoke("what's the weather in sf?")
    tool_node.invoke({"messages": [response_message]})
    ```

    1. 使用 `.bind_tools()` 将工具模式附加到聊天模型。

    ```
    {'messages': [ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='toolu_01Pnkgw5JeTRxXAU7tyHT4UW')]}
    ```

??? example "在调用工具的代理中使用"

    这是一个使用 `ToolNode` 从头开始创建调用工具的代理的示例。您还可以使用 LangGraph 的预建 [代理](../agents/agents.md)。

    ```python
    from langchain.chat_models import init_chat_model
    from langgraph.prebuilt import ToolNode
    from langgraph.graph import StateGraph, MessagesState, START, END
    
    def get_weather(location: str):
        """调用以获取当前天气。"""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."
    
    # highlight-next-line
    tool_node = ToolNode([get_weather])
    
    model = init_chat_model(model="claude-3-5-haiku-latest")
    # highlight-next-line
    model_with_tools = model.bind_tools([get_weather])
    
    def should_continue(state: MessagesState):
        messages = state["messages"]
        last_message = messages[-1]
        if last_message.tool_calls:
            return "tools"
        return END
    
    def call_model(state: MessagesState):
        messages = state["messages"]
        response = model_with_tools.invoke(messages)
        return {"messages": [response]}
    
    builder = StateGraph(MessagesState)
    
    # 定义我们将循环访问的两个节点
    builder.add_node("call_model", call_model)
    # highlight-next-line
    builder.add_node("tools", tool_node)
    
    builder.add_edge(START, "call_model")
    builder.add_conditional_edges("call_model", should_continue, ["tools", END])
    builder.add_edge("tools", "call_model")
    
    graph = builder.compile()
    
    graph.invoke({"messages": [{"role": "user", "content": "what's the weather in sf?"}]})
    ```
    
    ```
    {
        'messages': [
            HumanMessage(content="what's the weather in sf?"),
            AIMessage(
                content=[{'text': "I'll help you check the weather in San Francisco right now.", 'type': 'text'}, {'id': 'toolu_01A4vwUEgBKxfFVc5H3v1CNs', 'input': {'location': 'San Francisco'}, 'name': 'get_weather', 'type': 'tool_use'}],
                tool_calls=[{'name': 'get_weather', 'args': {'location': 'San Francisco'}, 'id': 'toolu_01A4vwUEgBKxfFVc5H3v1CNs', 'type': 'tool_call'}]
            ),
            ToolMessage(content="It's 60 degrees and foggy."),
            AIMessage(content="The current weather in San Francisco is 60 degrees and foggy. Typical San Francisco weather with its famous marine layer!")
        ]
    }
    ```

## 工具自定义

为了更精细地控制工具行为，请使用 `@tool` 装饰器。

### 参数描述

从文档字符串自动生成描述：

```python
# highlight-next-line
from langchain_core.tools import tool

# highlight-next-line
@tool("multiply_tool", parse_docstring=True)
def multiply(a: int, b: int) -> int:
    """Multiply two numbers.

    Args:
        a: First operand
        b: Second operand
    """
    return a * b
```

### 显式输入模式

使用 `args_schema` 定义模式：

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool

class MultiplyInputSchema(BaseModel):
    """Multiply two numbers"""
    a: int = Field(description="First operand")
    b: int = Field(description="Second operand")

# highlight-next-line
@tool("multiply_tool", args_schema=MultiplyInputSchema)
def multiply(a: int, b: int) -> int:
    return a * b
```

### 工具名称

使用第一个参数覆盖默认工具名称（函数名称）：

```python
from langchain_core.tools import tool

# highlight-next-line
@tool("multiply_tool")
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

## 上下文管理

LangGraph 中的工具有时需要上下文数据，例如不应由模型控制的运行时参数（例如，用户 ID 或会话详细信息）。LangGraph 提供三种管理此类上下文的方法：

| 类型                                    | 使用场景                           | 可变 | 生命周期                 |
|-----------------------------------------|------------------------------------------|---------|--------------------------|
| [配置](#configuration)         | 静态、不可变运行时数据           | ❌       | 单次调用        |
| [短期记忆](#short-term-memory) | 调用期间动态变化的数据           | ✅       | 单次调用        |
| [长期记忆](#long-term-memory)   | 持久化的跨会话数据           | ✅       | 跨多个会话 | 

### 配置

当您有工具所需的**不可变**运行时数据（例如用户标识符）时，请使用配置。您可以通过 [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) 在调用时传递这些参数，并在工具中访问它们：

```python
from langchain_core.tools import tool
from langchain_core.runnables import RunnableConfig

@tool
# highlight-next-line
def get_user_info(config: RunnableConfig) -> str:
    """检索基于用户 ID 的用户信息。"""
    user_id = config["configurable"].get("user_id")
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

# 具有代理的调用示例
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user info"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

??? example "扩展示例：在工具中访问配置"

    ```python
    from langchain_core.runnables import RunnableConfig
    from langchain_core.tools import tool
    from langgraph.prebuilt import create_react_agent
    
    def get_user_info(
        # highlight-next-line
        config: RunnableConfig,
    ) -> str:
        """检索状态中的用户名称。"""
        # highlight-next-line
        user_id = config["configurable"].get("user_id")
        return "User is John Smith" if user_id == "user_123" else "Unknown user"
    
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_user_info],
        state_schema=CustomState,
    )
    
    # 调用：从状态读取名称（初始为空）
    agent.invoke({"messages": "what's my name?"})
    ```

### 短期记忆

短期记忆维护在单次执行期间会**动态变化**的状态。

要**访问**（读取）图状态，您可以在工具中使用特殊的参数**注释** — [`InjectedState`][langgraph.prebuilt.InjectedState]：

```python
from typing import Annotated, NotRequired
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState

class CustomState(AgentState):
    # 在短期状态中的 user_name 字段
    user_name: NotRequired[str]

@tool
def get_user_name(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """从状态中检索当前用户名称。"""
    # 返回存储的名称，如果未设置，则返回默认值
    return state.get("user_name", "Unknown user")

# 示例代理设置
agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_name],
    state_schema=CustomState,
)

# 调用：从状态读取名称（初始为空）
agent.invoke({"messages": "what's my name?"})
```

使用返回 `Command` 的工具来**更新** `user_name` 并附加确认消息：

```python
from typing import Annotated
from langgraph.types import Command
from langchain_core.messages import ToolMessage
from langchain_core.tools import tool, InjectedToolCallId

@tool
def update_user_name(
    new_name: str,
    tool_call_id: Annotated[str, InjectedToolCallId]
) -> Command:
    """在短期记忆中更新用户名称。"""
    # highlight-next-line
    return Command(update={
        # highlight-next-line
        "user_name": new_name,
        # highlight-next-line
        "messages": [
            # highlight-next-line
            ToolMessage(f"Updated user name to {new_name}", tool_call_id=tool_call_id)
            # highlight-next-line
        ]
        # highlight-next-line
    })
```

!!! important

    如果您想使用返回 `Command` 并更新图状态的工具，您可以选择使用预建的 [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] / [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode] 组件，或者实现自己的工具执行节点，该节点收集工具返回的 `Command` 对象并返回一个列表，例如：
    
    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```

### 长期记忆

使用 [长期记忆](../concepts/memory.md#long-term-memory) 来跨会话存储用户特定或应用程序特定的数据。这对于聊天机器人等应用程序很有用，您希望在其中记住用户偏好或其他信息。

要使用长期记忆，您需要：

1. 为持久化数据跨调用[配置一个存储](memory/add-memory.md#add-long-term-memory)。
2. 使用 [`get_store`][langgraph.config.get_store] 函数从工具或提示中访问存储。

要**访问**存储中的信息：

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.tools import tool
from langgraph.graph import StateGraph
# highlight-next-line
from langgraph.config import get_store

@tool
def get_user_info(config: RunnableConfig) -> str:
    """查找用户信息。"""
    # 与 `builder.compile(store=store)` 
    # 或 `create_react_agent` 提供的相同
    # highlight-next-line
    store = get_store()
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"

builder = StateGraph(...)
...
graph = builder.compile(store=store)
```

??? example "访问长期记忆"

    ```python
    from langchain_core.runnables import RunnableConfig
    from langchain_core.tools import tool
    from langgraph.config import get_store
    from langgraph.prebuilt import create_react_agent
    from langgraph.store.memory import InMemoryStore
    
    # highlight-next-line
    store = InMemoryStore() # (1)!
    
    # highlight-next-line
    store.put(  # (2)!
        ("users",),  # (3)!
        "user_123",  # (4)!
        {
            "name": "John Smith",
            "language": "English",
        } # (5)!
    )

    @tool
    def get_user_info(config: RunnableConfig) -> str:
        """查找用户信息。"""
        # 与 `create_react_agent` 提供的相同
        # highlight-next-line
        store = get_store() # (6)!
        user_id = config["configurable"].get("user_id")
        # highlight-next-line
        user_info = store.get(("users",), user_id) # (7)!
        return str(user_info.value) if user_info else "Unknown user"
    
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_user_info],
        # highlight-next-line
        store=store # (8)!
    )
    
    # 运行代理
    agent.invoke(
        {"messages": [{"role": "user", "content": "look up user information"}]},
        # highlight-next-line
        config={"configurable": {"user_id": "user_123"}}
    )
    ```
    
    1. `InMemoryStore` 是一个在内存中存储数据的存储。在生产环境中，您通常会使用数据库或其他持久化存储。请参阅 [存储文档][../reference/store.md) 以获取更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产就绪的存储。
    2. 在此示例中，我们使用 `put` 方法将一些示例数据写入存储。有关更多详细信息，请参阅 [BaseStore.put][langgraph.store.base.BaseStore.put] API 参考。
    3. 第一个参数是命名空间。它用于将相关数据分组在一起。在此示例中，我们使用 `users` 命名空间来对用户数据进行分组。
    4. 命名空间内的键。此示例使用用户 ID 作为键。
    5. 我们要为给定用户存储的数据。
    6. `get_store` 函数用于访问存储。您可以从代码中的任何位置调用它，包括工具和提示。此函数返回创建代理时传递给代理的存储。
    7. `get` 方法用于从存储中检索数据。第一个参数是命名空间，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值以及关于该值元数据。
    8. `store` 被传递给代理。这使得代理在运行工具时可以访问存储。您还可以使用 `get_store` 函数从代码中的任何位置访问存储。

要**更新**存储中的信息：

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.tools import tool
from langgraph.graph import StateGraph
# highlight-next-line
from langgraph.config import get_store

@tool
def save_user_info(user_info: str, config: RunnableConfig) -> str:
    """保存用户信息。"""
    # 与 `builder.compile(store=store)` 
    # 或 `create_react_agent` 提供的相同
    # highlight-next-line
    store = get_store()
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    store.put(("users",), user_id, user_info)
    return "Successfully saved user info."

builder = StateGraph(...)
...
graph = builder.compile(store=store)
```

??? example "更新长期记忆"

    ```python
    from typing_extensions import TypedDict

    from langchain_core.tools import tool
    from langgraph.config import get_store
    from langgraph.prebuilt import create_react_agent
    from langgraph.store.memory import InMemoryStore
    
    store = InMemoryStore() # (1)!
    
    class UserInfo(TypedDict): # (2)!
        name: str

    @tool
    def save_user_info(user_info: UserInfo, config: RunnableConfig) -> str: # (3)!
        """保存用户信息。"""
        # 与 `create_react_agent` 提供的相同
        # highlight-next-line
        store = get_store() # (4)!
        user_id = config["configurable"].get("user_id")
        # highlight-next-line
        store.put(("users",), user_id, user_info) # (5)!
        return "Successfully saved user info."
    
    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[save_user_info],
        # highlight-next-line
        store=store
    )
    
    # 运行代理
    agent.invoke(
        {"messages": [{"role": "user", "content": "My name is John Smith"}]},
        # highlight-next-line
        config={"configurable": {"user_id": "user_123"}} # (6)!
    )
    
    # 您可以直接访问存储以获取值
    store.get(("users",), "user_123").value
    ```
    
    1. `InMemoryStore` 是一个在内存中存储数据的存储。在生产环境中，您通常会使用数据库或其他持久化存储。请参阅 [存储文档](../reference/store.md) 以获取更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产就绪的存储。
    2. `UserInfo` 类是一个 `TypedDict`，它定义了用户信息结构。LLM 将使用它根据模式格式化响应。
    3. `save_user_info` 函数是一个允许代理更新用户信息而不会干扰其内部状态的工具。这对于用户想要更新其个人资料信息的聊天应用程序很有用。
    4. `get_store` 函数用于访问存储。您可以从代码中的任何位置调用它，包括工具和提示。此函数返回创建代理时传递给代理的存储。
    5. `put` 方法用于将数据存储到存储中。第一个参数是命名空间，第二个参数是键。这将把用户信息存储在存储中。
    6. `user_id` 在配置中传递。这用于标识正在更新其信息的用户。

## 高级工具功能

### 即时返回

使用 `return_direct=True` 可在不执行额外逻辑的情况下直接返回工具的结果。

这对于不应触发进一步处理或工具调用的工具很有用，允许您直接将结果返回给用户。

```python
# highlight-next-line
@tool(return_direct=True)
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

??? example "扩展示例：在预建代理中使用 return_direct"

    ```python
    from langchain_core.tools import tool
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    @tool(return_direct=True)
    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[add]
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what's 3 + 5?"}]}
    )
    ```

!!! important "不带预建组件使用"

    如果您正在构建自定义工作流，并且不依赖于 `create_react_agent` 或 `ToolNode`，您还需要实现控制流来处理 `return_direct=True`。

### 强制使用工具

如果您需要强制使用特定工具，则需要在**模型**级别通过 `bind_tools` 方法中的 `tool_choice` 参数进行配置。

通过 tool_choice 强制使用特定工具：

```python
@tool(return_direct=True)
def greet(user_name: str) -> int:
    """Greet user."""
    return f"Hello {user_name}!"

tools = [greet]

configured_model = model.bind_tools(
    tools,
    # Force the use of the 'greet' tool
    # highlight-next-line
    tool_choice={"type": "tool", "name": "greet"}
)

```

??? example "扩展示例：在代理中强制使用工具"

    要强制代理使用特定工具，您可以在 `model.bind_tools()` 中设置 `tool_choice` 选项：

    ```python
    from langchain_core.tools import tool

    # highlight-next-line
    @tool(return_direct=True)
    def greet(user_name: str) -> int:
        """Greet user."""
        return f"Hello {user_name}!"

    tools = [greet]

    agent = create_react_agent(
        # highlight-next-line
        model=model.bind_tools(tools, tool_choice={"type": "tool", "name": "greet"}),
        tools=tools
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "Hi, I am Bob"}]}
    )
    ```

!!! Warning "避免无限循环"

    强制使用工具而不设置停止条件可能会导致无限循环。使用以下任一保护措施：

    - 使用 [`return_direct=True`](#immediate-return) 标记工具，在执行后结束循环。
    - 设置 [`recursion_limit`](../concepts/low_level.md#recursion-limit) 来限制执行步骤的数量。

!!! tip "工具选择配置"

    `tool_choice` 参数用于配置模型在决定调用工具时应使用的工具。当您想确保始终为特定任务调用某个工具，或者想覆盖模型选择工具的默认行为时，这很有用。

    请注意，并非所有模型都支持此功能，并且确切的配置可能因您使用的模型而异。

### 禁用并行调用

对于受支持的提供商，您可以通过 `model.bind_tools()` 方法设置 `parallel_tool_calls=False` 来禁用并行工具调用：

```python
model.bind_tools(
    tools, 
    # highlight-next-line
    parallel_tool_calls=False
)
```

??? example "扩展示例：在预建代理中禁用并行工具调用"

    ```python
    from langchain.chat_models import init_chat_model

    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    def multiply(a: int, b: int) -> int:
        """Multiply two numbers."""
        return a * b

    model = init_chat_model("anthropic:claude-3-5-sonnet-latest", temperature=0)
    tools = [add, multiply]
    agent = create_react_agent(
        # disable parallel tool calls
        # highlight-next-line
        model=model.bind_tools(tools, parallel_tool_calls=False),
        tools=tools
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what's 3 + 5 and 4 * 7?"}]}
    )
    ```

### 处理错误

LangGraph 通过预建的 [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode] 组件提供对工具执行的内置错误处理，该组件既可以独立使用，也可以用在预建代理中。

**默认情况下**，`ToolNode` 会捕获工具执行期间引发的异常，并将其作为 `ToolMessage` 对象返回，其中包含指示错误的 [status]。

```python
from langchain_core.messages import AIMessage
from langgraph.prebuilt import ToolNode

def multiply(a: int, b: int) -> int:
    if a == 42:
        raise ValueError("The ultimate error")
    return a * b

# 默认错误处理（默认启用）
tool_node = ToolNode([multiply])

message = AIMessage(
    content="",
    tool_calls=[{
        "name": "multiply",
        "args": {"a": 42, "b": 7},
        "id": "tool_call_id",
        "type": "tool_call"
    }]
)

result = tool_node.invoke({"messages": [message]})
```

输出：

```pycon
{'messages': [
    ToolMessage(
        content="Error: ValueError('The ultimate error')\n Please fix your mistakes.",
        name='multiply',
        tool_call_id='tool_call_id',
        status='error'
    )
]}
```

#### 禁用错误处理

要直接传播异常，请禁用错误处理：

```python
tool_node = ToolNode([multiply], handle_tool_errors=False)
```

禁用错误处理后，工具引发的异常将向上传播，需要显式管理。

#### 自定义错误消息

通过将 `handle_tool_errors` 设置为字符串来提供自定义错误消息：

```python
tool_node = ToolNode(
    [multiply],
    handle_tool_errors="Cannot use 42 as a first operand, please switch operands!"
)
```

示例输出：

```python
{'messages': [
    ToolMessage(
        content="Cannot use 42 as a first operand, please switch operands!",
        name='multiply',
        tool_call_id='tool_call_id',
        status='error'
    )
]}
```

#### 代理中的错误处理

预建代理中的错误处理（`create_react_agent`）利用了 `ToolNode`：

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[multiply]
)

# 默认错误处理
agent.invoke({"messages": [{"role": "user", "content": "what's 42 x 7?"}]})
```

要在预建代理中禁用或自定义错误处理，请显式传递已配置的 `ToolNode`：

```python
custom_tool_node = ToolNode(
    [multiply],
    handle_tool_errors="Cannot use 42 as a first operand!"
)

agent_custom = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=custom_tool_node
)

agent_custom.invoke({"messages": [{"role": "user", "content": "what's 42 x 7?"}]})
```

### 处理大量工具

随着可用工具数量的增长，您可能希望限制 LLM 的选择范围，以减少 token 消耗并帮助管理 LLM 推理中的错误来源。

为解决此问题，您可以通过在运行时使用语义搜索来动态调整可用于模型的工具。

有关现成实现的说明，请参阅 [`langgraph-bigtool`](https://github.com/langchain-ai/langgraph-bigtool) 预建库。

## 预建工具

### LLM 提供商工具

您可以通过将包含工具规范的字典传递给 `create_react_agent` 的 `tools` 参数来使用模型提供商的预建工具。例如，要使用 OpenAI 的 `web_search_preview` 工具：

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model="openai:gpt-4o-mini", 
    tools=[{"type": "web_search_preview"}]
)
response = agent.invoke(
    {"messages": ["What was a positive news story from today?"]}
)
```

请参阅您使用的特定模型的文档，了解可用的工具以及如何使用它们。

### LangChain 工具

此外，LangChain 还支持与 API、数据库、文件系统、Web 数据等交互的各种预建工具集成。这些工具扩展了代理的功能并实现了快速开发。

您可以在 [LangChain 集成目录](https://python.langchain.com/docs/integrations/tools/) 中浏览所有可用的集成。

一些常用的工具类别包括：

- **搜索**：Bing、SerpAPI、Tavily
- **代码解释器**：Python REPL、Node.js REPL
- **数据库**：SQL、MongoDB、Redis
- **Web 数据**：Web 抓取和浏览
- **API**：OpenWeatherMap、NewsAPI 等

这些集成可以使用上面示例中显示的相同 `tools` 参数进行配置和添加到您的代理中。