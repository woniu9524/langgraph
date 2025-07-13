# 构建一个基础聊天机器人

在本教程中，你将构建一个基础聊天机器人。这个聊天机器人是后续教程的基础，在后续教程中，你将逐步为其添加更复杂的功能，并在此过程中了解 LangGraph 的关键概念。让我们开始吧！🌟

## 先决条件

在开始本教程之前，请确保你可以访问支持
工具调用功能的 LLM，例如 [OpenAI](https://platform.openai.com/api-keys)、
[Anthropic](https://console.anthropic.com/settings/keys) 或
[Google Gemini](https://ai.google.dev/gemini-api/docs/api-key)。

## 1. 安装软件包

安装所需的软件包：

```bash
pip install -U langgraph langsmith
```

!!! tip

    注册 LangSmith，以便快速发现问题并提高 LangGraph 项目的性能。LangSmith 可让你利用跟踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用。有关如何开始的更多信息，请参阅 [LangSmith 文档](https://docs.smith.langchain.com)。

## 2. 创建一个 `StateGraph`

现在你可以使用 LangGraph 创建一个基础聊天机器人。这个聊天机器人将直接响应用户消息。

首先创建一个 `StateGraph`。`StateGraph` 对象将我们的聊天机器人结构定义为“状态机”。我们将添加 `nodes` 来表示聊天机器人可以调用的 LLM 和函数，以及 `edges` 来指定机器人应如何在这些函数之间进行转换。

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    # Messages 的类型为 "list"。
    # 注释中的 `add_messages` 函数定义了此状态键应如何更新
    # （在此例中，它会将消息追加到列表中，而不是覆盖它们）
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)
```

我们的图现在可以处理两个关键任务：

1. 每个 `node` 都可以接收当前 `State` 作为输入并输出对 state 的更新。
2. 感谢使用 `Annotated` 语法预先构建了 [`add_messages`](https://langchain-ai.github.io/langgraph/reference/graphs/?h=add+messages#add_messages) 函数，对 `messages` 的更新将追加到现有列表中，而不是覆盖它们。

------

!!! tip "概念"

    在定义图时，第一步是定义其 `State`。`State` 包含图的 schema 和处理 state 更新的 [reducer 函数](https://langchain-ai.github.io/langgraph/concepts/low_level/#reducers)。在我们的示例中，`State` 是一个 `TypedDict`，包含一个键：`messages`。[`add_messages`](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.message.add_messages) reducer 函数用于将新消息追加到列表中，而不是覆盖它。没有 reducer 注释的键将覆盖之前的earlier值。要了解有关 state、reducers 和相关概念的更多信息，请参阅 [LangGraph 参考文档](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.message.add_messages)。

## 3. 添加一个节点

接下来，添加一个“`chatbot`”节点。**节点**代表工作单元，通常是普通的 Python 函数。

首先，让我们选择一个聊天模型：

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->


现在我们可以将聊天模型合并到一个简单的节点中：

```python

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一个参数是唯一的节点名称
# 第二个参数是每次调用节点时将调用该函数或对象。
graph_builder.add_node("chatbot", chatbot)
```

**请注意** `chatbot` 节点函数如何将当前 `State` 作为输入，并返回一个字典，该字典在“messages”键下包含一个更新的 `messages` 列表。这是所有 LangGraph 节点函数的基本模式。

我们 `State` 中的 `add_messages` 函数将把 LLM 的响应消息追加到 state 中已有的任何消息之后。

## 4. 添加一个 `entry` 点

添加一个 `entry` 点，告知图每次运行时**从何处开始工作**：

```python
graph_builder.add_edge(START, "chatbot")
```

## 5. 添加一个 `exit` 点

添加一个 `exit` 点，指示**图应在何处完成执行**。这对于更复杂的流程很有用，但即使在这个简单的图中，添加一个结束节点也能提高清晰度。

```python
graph_builder.add_edge("chatbot", END)
```
这会告诉图在运行完 chatbot 节点后终止。

## 6. 编译图

在运行图之前，我们需要编译它。我们可以通过在图构建器上调用 `compile()` 来实现。这将创建一个 `CompiledStateGraph`，我们可以在其上调用 state。

```python
graph = graph_builder.compile()
```

## 7. 可视化图 (可选)

你可以使用 `get_graph` 方法和其中一个“draw”方法（例如 `draw_ascii` 或 `draw_png`）来可视化图。`draw` 方法每个都需要额外的依赖项。

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # 这需要一些额外的依赖项，并且是可选的
    pass
```

![basic chatbot diagram](basic-chatbot.png)


## 8. 运行聊天机器人

现在运行聊天机器人！

!!! tip

    你可以随时通过键入 `quit`、`exit` 或 `q` 来退出聊天循环。

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)


while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break
        stream_graph_updates(user_input)
    except:
        # 如果 input() 不可用则回退
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```

```
Assistant: LangGraph 是一个库，旨在帮助使用语言模型构建有状态的多代理应用程序。它提供了创建工作流和状态机以协调多个 AI 代理或语言模型交互的工具。LangGraph 构建在 LangChain 之上，利用其组件，同时增加了基于图的协调功能。它对于开发超越简单查询-响应交互的更复杂、有状态的 AI 应用程序特别有用。
Goodbye!
```

**恭喜！** 你已经使用 LangGraph 构建了你的第一个聊天机器人。这个机器人可以通过接收用户输入并使用 LLM 生成响应来进行基本对话。你可以检查上面的调用 [LangSmith Trace](https://smith.langchain.com/public/7527e308-9502-4894-b347-f34385740d5a/r)。

下面是本教程的完整代码：

```python
from typing import Annotated

from langchain.chat_models import init_chat_model
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)


llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")


def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一个参数是唯一的节点名称
# 第二个参数是每次调用节点时将调用该函数或对象。
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
graph = graph_builder.compile()
```

## 接下来的步骤

你可能已经注意到，该机器人的知识仅限于其训练数据。在下一部分中，我们将[添加一个网络搜索工具](./2-add-tools.md)，以扩展机器人的知识并使其功能更强大。