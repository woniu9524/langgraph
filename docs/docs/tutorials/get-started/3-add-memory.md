# 添加记忆

聊天机器人现在可以[使用工具](./2-add-tools.md)来回答用户问题了，但它无法记住之前交互的上下文。这限制了它进行连贯的多轮对话的能力。

LangGraph 通过**持久化检查点**解决了这个问题。如果你在编译图时提供了一个 `checkpointer`，并在调用图时提供了一个 `thread_id`，LangGraph 会在每一步后自动保存状态。当你使用相同的 `thread_id` 再次调用图时，图会加载其保存的状态，从而让聊天机器人可以从上次中断的地方继续。

稍后我们会看到，**检查点**比简单的聊天记忆 _强大得多_——它允许你在任何时候保存和恢复复杂状态，以实现错误恢复、人工审核工作流、时间旅行交互等。但首先，让我们添加检查点以启用多轮对话。

!!! note

    本教程建立在 [添加工具](./2-add-tools.md) 的基础上。

## 1. 创建一个 `MemorySaver` 检查点

创建一个 `MemorySaver` 检查点：

```python
from langgraph.checkpoint.memory import MemorySaver

memory = MemorySaver()
```

这是一个内存中的检查点，对于本教程来说很方便。但是，在生产环境中，你可能会将其更改为使用 `SqliteSaver` 或 `PostgresSaver` 并连接数据库。

## 2. 编译图

使用提供的检查点来编译图，它将在图处理每个节点时检查点 `State`：

```python
graph = graph_builder.compile(checkpointer=memory)
```

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

## 3. 与你的聊天机器人互动

现在你可以与你的机器人互动了！

1. 选择一个 thread 来作为本次对话的 key。

    ```python
    config = {"configurable": {"thread_id": "1"}}
    ```

2. 调用你的聊天机器人：

    ```python
    user_input = "Hi there! My name is Will."

    # Config 是 stream() 或 invoke() 的 **第二个位置参数**！
    events = graph.stream(
        {"messages": [{"role": "user", "content": user_input}]},
        config,
        stream_mode="values",
    )
    for event in events:
        event["messages"][-1].pretty_print()
    ```

    ```
    ================================ Human Message =================================

    Hi there! My name is Will.
    ================================== Ai Message ==================================

    Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
    ```

    !!! note 

        在调用我们的图时，Config 作为了 **第二个位置参数**。重要的是，它没有嵌套在图的输入（`{'messages': []}`）中。

## 4. 提出后续问题

提出一个后续问题：

```python
user_input = "Remember my name?"

# Config 是 stream() 或 invoke() 的 **第二个位置参数**！
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.
```

**请注意**，我们没有使用外部列表来存储记忆：所有内容都由检查点处理！你可以查看此 [LangSmith trace](https://smith.langchain.com/public/29ba22b5-6d40-4fbe-8d27-b369e3329c84/r) 中的完整执行过程，了解正在发生什么。

不相信我？尝试使用不同的 config 来验证。

```python
# 唯一的区别是我们在这里将 `thread_id` 改为 "2" 而不是 "1"
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    # highlight-next-line
    {"configurable": {"thread_id": "2"}},
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

I apologize, but I don't have any previous context or memory of your name. As an AI assistant, I don't retain information from past conversations. Each interaction starts fresh. Could you please tell me your name so I can address you properly in this conversation?
```

**请注意**，我们所做的**唯一**改变是修改了 Config 中的 `thread_id`。请参阅此调用的 [LangSmith trace](https://smith.langchain.com/public/51a62351-2f0a-4058-91cc-9996c5561428/r) 进行比较。

## 5. 检查状态

现在，我们已经在两个不同的 thread 中进行了几次检查点。但检查点中包含什么？要随时检查给定配置的图的 `state`，请调用 `get_state(config)`。

```python
snapshot = graph.get_state(config)
snapshot
```

```
StateSnapshot(values={'messages': [HumanMessage(content='Hi there! My name is Will.', additional_kwargs={}, response_metadata={}, id='8c1ca919-c553-4ebf-95d4-b59a2d61e078'), AIMessage(content="Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?", additional_kwargs={}, response_metadata={'id': 'msg_01WTQebPhNwmMrmmWojJ9KXJ', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 405, 'output_tokens': 32}}, id='run-58587b77-8c82-41e6-8a90-d62c444a261d-0', usage_metadata={'input_tokens': 405, 'output_tokens': 32, 'total_tokens': 437}), HumanMessage(content='Remember my name?', additional_kwargs={}, response_metadata={}, id='daba7df6-ad75-4d6b-8057-745881cea1ca'), AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-93e0-6acc-8004-f2ac846575d2'}}, metadata={'source': 'loop', 'writes': {'chatbot': {'messages': [AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}}, 'step': 4, 'parents': {}}, created_at='2024-09-27T19:30:10.820758+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-859f-6206-8003-e1bd3c264b8f'}}, tasks=())
```

```python
snapshot.next  # (因为图在此轮结束了，所以 `next` 是空的。如果你获取的是图调用中的状态，next 会告诉你下一个将要执行的节点)
```

上面的快照包含当前状态值、相应的配置以及要处理的 `next` 节点。在我们的例子中，图已达到 `END` 状态，因此 `next` 为空。

**恭喜！** 借助 LangGraph 的检查点系统，你的聊天机器人现在可以在会话之间维护对话状态。这为更自然、更具上下文的交互开启了令人兴奋的可能性。LangGraph 的检查点甚至可以处理**任意复杂的图状态**，这比简单的聊天记忆更具表现力和强大性。
  
查看下面的代码片段，回顾本教程中的图：

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python hl_lines="36 37"
from typing import Annotated

from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)
tools = [tool]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.set_entry_point("chatbot")
memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

## 下一步

在下一个教程中，你将[向聊天机器人添加人工审核](./4-human-in-the-loop.md)，以处理在某些情况下需要指导或验证才能继续进行的情况。