# 运行时重建图

有时，您可能需要使用不同的配置重建图以进行新的运行。例如，根据配置，您可能需要使用不同的图状态或图结构。本指南将向您展示如何实现这一点。

!!! note "注意"
    在大多数情况下，基于配置定制行为应由单个图处理，其中每个节点都可以读取配置并根据该配置更改其行为。

## 先决条件

首先，请务必查看 [此操作指南](./setup.md) 中关于为部署设置应用的内容。

## 定义图

假设您有一个应用，其中包含调用 LLM 并将响应返回给用户的简单图。该应用的文件目录如下所示：

```
my-app/
|-- requirements.txt
|-- .env
|-- openai_agent.py     # your graph code
```

其中图定义在 `openai_agent.py` 中。

### 无需重建

在标准的 LangGraph API 配置中，服务器使用在 `openai_agent.py` 顶层定义的已编译图实例，如下所示：

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph

model = ChatOpenAI(temperature=0)

graph_workflow = MessageGraph()

graph_workflow.add_node("agent", model)
graph_workflow.add_edge("agent", END)
graph_workflow.add_edge(START, "agent")

agent = graph_workflow.compile()
```

要使服务器意识到您的图，您需要在 LangGraph API 配置 (`langgraph.json`) 中指定包含 `CompiledStateGraph` 实例的变量的路径，例如：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:agent",
    },
    "env": "./.env"
}
```

### 重建

为了让您的图在每次运行时使用自定义配置进行重建，您需要重写 `openai_agent.py`，以便提供一个接收配置并返回图（或已编译图）实例的_函数_。假设我们想为用户 ID '1' 返回我们现有的图，为其他用户返回一个工具调用代理。我们可以按如下方式修改 `openai_agent.py`：

```python
from typing import Annotated
from typing_extensions import TypedDict
from langchain_openai import ChatOpenAI
from langgraph.graph import END, START, MessageGraph
from langgraph.graph.state import StateGraph
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode
from langchain_core.tools import tool
from langchain_core.messages import BaseMessage
from langchain_core.runnables import RunnableConfig


class State(TypedDict):
    messages: Annotated[list[BaseMessage], add_messages]


model = ChatOpenAI(temperature=0)

def make_default_graph():
    """创建一个简单的 LLM 代理"""
    graph_workflow = StateGraph(State)
    def call_model(state):
        return {"messages": [model.invoke(state["messages"])]}

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_edge("agent", END)
    graph_workflow.add_edge(START, "agent")

    agent = graph_workflow.compile()
    return agent


def make_alternative_graph():
    """创建一个工具调用代理"""

    @tool
    def add(a: float, b: float):
        """将两个数字相加。"""
        return a + b

    tool_node = ToolNode([add])
    model_with_tools = model.bind_tools([add])
    def call_model(state):
        return {"messages": [model_with_tools.invoke(state["messages"])]}

    def should_continue(state: State):
        if state["messages"][-1].tool_calls:
            return "tools"
        else:
            return END

    graph_workflow = StateGraph(State)

    graph_workflow.add_node("agent", call_model)
    graph_workflow.add_node("tools", tool_node)
    graph_workflow.add_edge("tools", "agent")
    graph_workflow.add_edge(START, "agent")
    graph_workflow.add_conditional_edges("agent", should_continue)

    agent = graph_workflow.compile()
    return agent


# 这是图创建函数，它将根据提供的配置决定构建哪个图
def make_graph(config: RunnableConfig):
    user_id = config.get("configurable", {}).get("user_id")
    # 根据用户 ID 路由到不同的图状态/结构
    if user_id == "1":
        return make_default_graph()
    else:
        return make_alternative_graph()
```

最后，您需要在 `langgraph.json` 中指定图创建函数（`make_graph`）的路径：

```
{
    "dependencies": ["."],
    "graphs": {
        "openai_agent": "./openai_agent.py:make_graph",
    },
    "env": "./.env"
}
```

有关 LangGraph API 配置文件，请参阅 [此处](../reference/cli.md#configuration-file)。