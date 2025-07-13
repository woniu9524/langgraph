# 添加人工干预控制

代理可能并不可靠，并且可能需要人工输入才能成功完成任务。同样，对于某些操作，您可能希望在运行前要求人工批准，以确保一切都按预期运行。

LangGraph 的 [持久性](../../concepts/persistence.md)层支持**人工干预**工作流，允许根据用户反馈暂停和恢复执行。此功能的主要接口是 [`interrupt`](../../how-tos/human_in_the_loop/add-human-in-the-loop.md) 函数。在节点内调用 `interrupt` 将会暂停执行。通过传入一个 [Command](../../concepts/low_level.md#command)，可以与来自人工的新输入一起恢复执行。`interrupt` 在功能上与 Python 的内置 `input()` 类似，[但有一些注意事项](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。

!!! note

    本教程建立在 [添加记忆](./3-add-memory.md) 的基础上。

## 1. 添加 `human_assistance` 工具

从 [为聊天机器人添加记忆](./3-add-memory.md) 教程的现有代码开始，将 `human_assistance` 工具添加到聊天机器人中。此工具使用 `interrupt` 从人工接收信息。

我们先选择一个聊天模型：

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

现在，我们可以将其与一个额外的工具一起整合到我们的 `StateGraph` 中：

```python hl_lines="12 19 20 21 22 23"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    # Because we will be interrupting during tool execution,
    # we disable parallel tool calling to avoid repeating any
    # tool invocations when we resume.
    assert len(message.tool_calls) <= 1
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
```

!!! tip

    有关人工干预工作流的更多信息和示例，请参阅 [人工干预](../../concepts/human_in_the_loop.md)。

## 2. 编译图表

我们像以前一样编译图表并添加检查点：

```python
memory = MemorySaver()

graph = graph_builder.compile(checkpointer=memory)
```

## 3. 可视化图表（可选）

可视化图表，您将看到与之前相同的布局 – 只是添加了工具！

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

![chatbot-with-tools-diagram](chatbot-with-tools.png)

## 4. 提示聊天机器人

现在，用一个会触发新的 `human_assistance` 工具的问题来提示聊天机器人：

```python
user_input = "I need some expert guidance for building an AI agent. Could you request assistance for me?"
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I need some expert guidance for building an AI agent. Could you request assistance for me?
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
```

聊天机器人生成了一个工具调用，但随后执行被中断了。如果检查图状态，您会发现它在工具节点处停止了：

```python
snapshot = graph.get_state(config)
snapshot.next
```

```
('tools',)
```

!!! info 额外信息

    仔细查看 `human_assistance` 工具：

    ```python
    @tool
    def human_assistance(query: str) -> str:
        """Request assistance from a human."""
        human_response = interrupt({"query": query})
        return human_response["data"]
    ```

    与 Python 的内置 `input()` 函数类似，在工具内调用 `interrupt` 将会暂停执行。进度会根据 [检查点](../../concepts/persistence.md#checkpointer-libraries) 进行持久化；因此，如果使用 Postgres 进行持久化，只要数据库保持运行，它就可以在任何时候恢复。在此示例中，它使用内存检查点进行持久化，只要 Python 内核正在运行，就可以在任何时候恢复。

## 5. 恢复执行

要恢复执行，请传递一个包含工具预期数据的 [`Command`](../../concepts/low_level.md#command) 对象。此数据的格式可根据需要进行自定义。在此示例中，使用带有 `"data"` 键的字典：

```python
human_response = (
    "We, the experts are here to help! We'd recommend you check out LangGraph to build your agent."
    " It's much more reliable and extensible than simple autonomous agents."
)

human_command = Command(resume={"data": human_response})

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
================================= Tool Message =================================
Name: human_assistance

We, the experts are here to help! We'd recommend you check out LangGraph to build your agent. It's much more reliable and extensible than simple autonomous agents.
================================== Ai Message ==================================

Thank you for your patience. I've received some expert advice regarding your request for guidance on building an AI agent. Here's what the experts have suggested:

The experts recommend that you look into LangGraph for building your AI agent. They mention that LangGraph is a more reliable and extensible option compared to simple autonomous agents.

LangGraph is likely a framework or library designed specifically for creating AI agents with advanced capabilities. Here are a few points to consider based on this recommendation:

1. Reliability: The experts emphasize that LangGraph is more reliable than simpler autonomous agent approaches. This could mean it has better stability, error handling, or consistent performance.

2. Extensibility: LangGraph is described as more extensible, which suggests that it probably offers a flexible architecture that allows you to easily add new features or modify existing ones as your agent's requirements evolve.

3. Advanced capabilities: Given that it's recommended over "simple autonomous agents," LangGraph likely provides more sophisticated tools and techniques for building complex AI agents.
...
2. Look for tutorials or guides specifically focused on building AI agents with LangGraph.
3. Check if there are any community forums or discussion groups where you can ask questions and get support from other developers using LangGraph.

If you'd like more specific information about LangGraph or have any questions about this recommendation, please feel free to ask, and I can request further assistance from the experts.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

输入已接收并作为工具消息处理。查看此调用的 [LangSmith 跟踪](https://smith.langchain.com/public/9f0f87e3-56a7-4dde-9c76-b71675624e91/r)以查看上一次调用中执行的确切工作。请注意，状态会在第一步加载，以便我们的聊天机器人可以从中断处继续。

**恭喜！** 您已使用 `interrupt` 为聊天机器人添加了人工干预执行，从而可以在需要时进行人工监督和干预。这为您的 AI 系统可以创建的用户界面打开了可能性。由于您已经添加了**检查点**，只要底层持久性层正在运行，图就可以**无限期**暂停并在任何时候恢复，如同什么都没有发生过一样。

查看下面的代码片段以回顾本教程中的图：

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = MemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

## 后续步骤

到目前为止，教程示例都依赖于一个简单的状态，其中包含一个条目：消息列表。您可以使用这个简单的状态走得很远，但如果您想定义复杂行为而不依赖于消息列表，您可以 [向状态添加其他字段](./5-customize-state.md)。