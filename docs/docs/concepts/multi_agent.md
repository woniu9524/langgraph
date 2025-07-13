---
search:
  boost: 2
---

# 多智能体系统

[智能体](./agentic_concepts.md#agent-architectures)是_一个使用 LLM 来决定应用程序控制流程的系统_。随着您开发这些系统，它们可能会随着时间的推移变得更加复杂，导致管理和扩展变得更加困难。例如，您可能会遇到以下问题：

- 智能体可用的工具过多，难以做出调用下一个工具的明智决策
- 上下文过于复杂，单个智能体难以跟踪
- 系统需要多个专业领域（例如，规划师、研究员、数学专家等）

为了解决这些问题，您可以考虑将应用程序分解为多个更小、独立的智能体，并将它们组合成一个**多智能体系统**。这些独立的智能体可以很简单，只是一个提示和一个 LLM 调用，也可以很复杂，例如 [ReAct](./agentic_concepts.md#tool-calling-agent) 智能体（甚至更复杂！）。

使用多智能体系统的主要好处包括：

- **模块化**: 独立的智能体使得开发、测试和维护智能体系统更加容易。
- **专业化**: 您可以创建专注于特定领域的专家智能体，这有助于提高整个系统的性能。
- **控制**: 您可以显式控制智能体如何通信（而不是依赖函数调用）。

## 多智能体架构

![](./img/multi_agent/architectures.png)

在多智能体系统中连接智能体有几种方式：

- **网络 (Network)**: 每个智能体都可以与 [所有其他智能体](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/multi-agent-collaboration/) 通信。任何智能体都可以决定接下来调用哪个其他智能体。
- **Supervisor**: 每个智能体都与一个 [Supervisor](./tutorials/multi_agent/agent_supervisor.md) 智能体通信。Supervisor 智能体负责决定接下来应该调用哪个智能体。
- **Supervisor (tool-calling)**: 这是 Supervisor 架构的一个特例。单个智能体可以表示为工具。在这种情况下，Supervisor 智能体使用支持工具调用的 LLM 来决定调用哪个智能体工具以及传递给这些智能体的参数。
- **Hierarchical (层级式)**: 您可以通过 [Supervisor of Supervisors](https://langchain-ai.github.io/langgraph/tutorials/multi_agent/hierarchical_agent_teams/) 来定义一个多智能体系统。这是 Supervisor 架构的泛化，允许更复杂的控制流。
- **Custom multi-agent workflow (自定义多智能体工作流)**: 每个智能体仅与一部分智能体通信。流程的某些部分是确定的，只有少数智能体可以决定接下来调用哪个其他智能体。

### Handoffs (交接)

在多智能体架构中，智能体可以表示为图节点。每个智能体节点执行其步骤，并决定是完成执行还是路由到另一个智能体，包括可能路由到自身（例如，循环运行）。多智能体交互中的一个常见模式是 **handoffs (交接)**，即一个智能体将控制权“交接”给另一个智能体。交接允许您指定：

- __destination__: 要导航到的目标智能体（例如，节点的名称）
- __payload__: [要传递给该智能体的信息](#communication-and-state-management)（例如，状态更新）

要在 LangGraph 中实现交接，智能体节点可以返回 [`Command`](./low_level.md#command) 对象，该对象允许您同时处理控制流和状态更新：

```python
def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # 路由/停止的条件可以是任何内容，例如 LLM 工具调用 / 结构化输出等
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    return Command(
        # 指定下一个要调用的智能体
        goto=goto,
        # 更新图状态
        update={"my_state_key": "my_state_value"}
    )
```

在更复杂的场景中，每个智能体节点本身就是一个图（即一个 [子图](./subgraphs.md)），一个智能体子图中的节点可能想导航到另一个智能体。例如，如果您有两个智能体 `alice` 和 `bob`（父图中的子图节点），并且 `alice` 需要导航到 `bob`，您可以在 `Command` 对象中将 `graph=Command.PARENT` 设置为：

```python
def some_node_inside_alice(state):
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # 指定要导航到的图（默认为当前图）
        graph=Command.PARENT,
    )
```

!!! note
    如果您需要为使用 `Command(graph=Command.PARENT)` 进行通信的子图提供可视化支持，您需要将它们包装在一个带有 `Command` 注释的节点函数中，例如，而不是：

    ```python
    builder.add_node(alice)
    ```

    您需要这样做：

    ```python
    def call_alice(state) -> Command[Literal["bob"]]:
        return alice.invoke(state)

    builder.add_node("alice", call_alice)
    ```

#### Handoffs as tools (交接作为工具)

最常见的智能体类型之一是 [工具调用智能体](../agents/overview.md)。对于这些类型的智能体，一个常见的模式是将交接包装在工具调用中，例如：

```python
from langchain_core.tools import tool

@tool
def transfer_to_bob():
    """Transfer to bob."""
    return Command(
        # 要转到的智能体（节点）的名称
        goto="bob",
        # 要发送给智能体的数据
        update={"my_state_key": "my_state_value"},
        # 指示 LangGraph，我们需要导航到
        # 父图中的智能体节点
        graph=Command.PARENT,
    )
```

这是从工具更新图状态的一个特例，除了状态更新之外，还包括了控制流。

!!! important

    如果您想使用返回 `Command` 的工具，您可以选择预构建的 [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] / [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode] 组件，或者实现自己的工具执行节点，该节点收集工具返回的 `Command` 对象并返回它们的列表，例如：
    
    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```

现在让我们仔细看看不同的多智能体架构。

### Network (网络)

在此架构中，智能体被定义为图节点。每个智能体都可以与所有其他智能体通信（多对多连接），并可以决定接下来调用哪个智能体。此架构适用于没有清晰智能体层级结构或没有特定智能体调用顺序的问题。


```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # 您可以将状态的相关部分传递给 LLM（例如 state["messages"]）
    # 以确定接下来调用哪个智能体。一个常见的模式是使用结构化输出来调用模型
    # （例如，强制它返回一个带有 "next_agent" 字段的输出）
    response = model.invoke(...)
    # 根据 LLM 的决定路由到其中一个智能体或退出
    # 如果 LLM 返回 "__end__"，图将完成执行
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_2(state: MessagesState) -> Command[Literal["agent_1", "agent_3", END]]:
    response = model.invoke(...)
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

def agent_3(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    ...
    return Command(
        goto=response["next_agent"],
        update={"messages": [response["content"]]},
    )

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
builder.add_node(agent_3)

builder.add_edge(START, "agent_1")
network = builder.compile()
```

### Supervisor (主管)

在此架构中，我们将智能体定义为节点，并添加一个 Supervisor 节点（LLM），该节点决定应该调用哪个智能体节点。我们使用 [`Command`](./low_level.md#command) 根据 Supervisor 的决定将执行路由到适当的智能体节点。此架构也很适合并行运行多个智能体或使用 [map-reduce](./how-tos/graph-api.md#map-reduce-and-the-send-api) 模式。

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    # 您可以将状态的相关部分传递给 LLM（例如 state["messages"]）
    # 以确定接下来调用哪个智能体。一个常见的模式是使用结构化输出来调用模型
    # （例如，强制它返回一个带有 "next_agent" 字段的输出）
    response = model.invoke(...)
    # 根据 Supervisor 的决定路由到其中一个智能体或退出
    # 如果 Supervisor 返回 "__end__"，图将完成执行
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    # 您可以将状态的相关部分传递给 LLM（例如 state["messages"]）
    # 并添加任何额外的逻辑（不同的模型、自定义提示、结构化输出等）
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

def agent_2(state: MessagesState) -> Command[Literal["supervisor"]]:
    response = model.invoke(...)
    return Command(
        goto="supervisor",
        update={"messages": [response]},
    )

builder = StateGraph(MessagesState)
builder.add_node(supervisor)
builder.add_node(agent_1)
builder.add_node(agent_2)

builder.add_edge(START, "supervisor")

supervisor = builder.compile()
```

请查看此 [教程](../tutorials/multi_agent/agent_supervisor.md) 以获取 Supervisor 多智能体架构的示例。

### Supervisor (tool-calling) (主管 - 工具调用)

在此 [Supervisor](#supervisor) 架构的变体中，我们定义了一个 Supervisor [智能体](./agentic_concepts.md#agent-architectures)，该智能体负责调用子智能体。子智能体作为工具暴露给 Supervisor，Supervisor 智能体决定下一个调用哪个工具。Supervisor 智能体遵循 [标准实现](./agentic_concepts.md#tool-calling-agent)，即一个 LLM 在循环中运行，调用工具直至决定停止。

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# 这是将作为工具调用的智能体函数
# 注意您可以将状态通过 InjectedState 注释传递给工具
def agent_1(state: Annotated[dict, InjectedState]):
    # 您可以将状态的相关部分传递给 LLM（例如 state["messages"]）
    # 并添加任何额外的逻辑（不同的模型、自定义提示、结构化输出等）
    response = model.invoke(...)
    # 以字符串格式返回 LLM 响应（预期的工具响应格式）
    # 这将由预构建的 create_react_agent（Supervisor）自动转换为 ToolMessage
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# 构建带工具调用的 Supervisor 的最简单方法是使用预构建的 ReAct 智能体图
# 该图由一个工具调用 LLM 节点（即 Supervisor）和一个工具执行节点组成
supervisor = create_react_agent(model, tools)
```

### Hierarchical (层级式)

随着您在系统中添加更多智能体，Supervisor 可能难以管理所有智能体。Supervisor 可能开始对调用哪个智能体做出糟糕的决策，或者上下文可能变得过于复杂，以至于单个 Supervisor 无法跟踪。换句话说，您会遇到最初促使采用多智能体架构的 same problems。

为了解决这个问题，您可以设计一个 _分层_ 系统。例如，您可以创建由各个 Supervisor 管理的独立、专业化的智能体团队，并由一个顶层 Supervisor 来管理这些团队。

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
model = ChatOpenAI()

# 定义团队 1（与上面的单一 Supervisor 示例相同）

def team_1_supervisor(state: MessagesState) -> Command[Literal["team_1_agent_1", "team_1_agent_2", END]]:
    response = model.invoke(...)
    return Command(goto=response["next_agent"])

def team_1_agent_1(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

def team_1_agent_2(state: MessagesState) -> Command[Literal["team_1_supervisor"]]:
    response = model.invoke(...)
    return Command(goto="team_1_supervisor", update={"messages": [response]})

team_1_builder = StateGraph(Team1State)
team_1_builder.add_node(team_1_supervisor)
team_1_builder.add_node(team_1_agent_1)
team_1_builder.add_node(team_1_agent_2)
team_1_builder.add_edge(START, "team_1_supervisor")
team_1_graph = team_1_builder.compile()

# 定义团队 2（与上面的单一 Supervisor 示例相同）
class Team2State(MessagesState):
    next: Literal["team_2_agent_1", "team_2_agent_2", "__end__"]

def team_2_supervisor(state: Team2State):
    ...

def team_2_agent_1(state: Team2State):
    ...

def team_2_agent_2(state: Team2State):
    ...

team_2_builder = StateGraph(Team2State)
...
team_2_graph = team_2_builder.compile()


# 定义顶层 Supervisor

builder = StateGraph(MessagesState)
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    # 您可以将状态的相关部分传递给 LLM（例如 state["messages"]）
    # 以确定接下来调用哪个团队。一个常见的模式是使用结构化输出来调用模型
    # （例如，强制它返回一个带有 "next_team" 字段的输出）
    response = model.invoke(...)
    # 根据 Supervisor 的决定路由到其中一个团队或退出
    # 如果 Supervisor 返回 "__end__"，图将完成执行
    return Command(goto=response["next_team"])

builder = StateGraph(MessagesState)
builder.add_node(top_level_supervisor)
builder.add_node("team_1_graph", team_1_graph)
builder.add_node("team_2_graph", team_2_graph)
builder.add_edge(START, "top_level_supervisor")
builder.add_edge("team_1_graph", "top_level_supervisor")
builder.add_edge("team_2_graph", "top_level_supervisor")
graph = builder.compile()
```

### Custom multi-agent workflow (自定义多智能体工作流)

在此架构中，我们将单个智能体添加为图节点，并提前定义智能体调用的顺序，在一个自定义工作流中。在 LangGraph 中，工作流可以通过两种方式定义：

- **Explicit control flow (normal edges) (显式控制流 - 普通边)**: LangGraph 允许您通过[普通图边](./low_level.md#normal-edges)显式定义应用程序的控制流（即智能体通信的顺序）。这是此架构中最确定的变体——我们总是提前知道哪个智能体将被调用。

- **Dynamic control flow (Command) (动态控制流 - Command)**: 在 LangGraph 中，您可以允许 LLM 控制应用程序控制流的部分。这可以通过使用[`Command`](./low_level.md#command)来实现。其中的一个特例是[Supervisor 工具调用](#supervisor-tool-calling)架构。在这种情况下，为 Supervisor 智能体提供支持的工具调用 LLM 将决定调用工具（智能体）的顺序。

```python
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START

model = ChatOpenAI()

def agent_1(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

def agent_2(state: MessagesState):
    response = model.invoke(...)
    return {"messages": [response]}

builder = StateGraph(MessagesState)
builder.add_node(agent_1)
builder.add_node(agent_2)
# 清晰地定义流程
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

## Communication and state management (通信和状态管理)

构建多智能体系统时最重要的事情是确定智能体如何通信。

一种常见、通用的智能体通信方式是通过消息列表。这引出了以下问题：

- 智能体是通过[**交接还是工具调用**](#handoffs-vs-tool-calls)进行通信？
- [**从一个智能体传递到下一个智能体**](#message-passing-between-agents)的消息是什么？
- [**如何在消息历史中表示交接**](#representing-handoffs-in-message-history)？
- 如何[**管理子智能体的状态**](#state-management-for-subagents)？

此外，如果您正在处理更复杂的智能体或希望将单个智能体的状态与多智能体系统的状态分开，则可能需要使用[**不同的状态模式**](#using-different-state-schemas)。

### Handoffs vs tool calls (交接 vs 工具调用)

在智能体之间传递的“有效载荷”是什么？在上面讨论的大多数架构中，智能体通过[交接](#handoffs)进行通信，并将[图状态](./low_level.md#state)作为交接有效载荷的一部分进行传递。特别是，智能体将消息列表作为图状态的一部分进行传递。在带工具调用的 Supervisor 的情况下，有效载荷是工具调用参数。

![](./img/multi_agent/request.png)

### Message passing between agents (智能体之间的消息传递)

智能体通信最常见的方式是通过共享状态通道，通常是消息列表。这假定状态中总是有至少一个通道（键）被智能体共享（例如 `messages`）。通过共享消息列表进行通信时，还有一个额外的考虑因素：智能体是应[共享完整的思考过程](#sharing-full-thought-process)还是仅[共享最终结果](#sharing-only-final-results)？

![](./img/multi_agent/response.png)

#### Sharing full thought process (共享完整的思考过程)

智能体可以与所有其他智能体**共享其完整的思考过程**（即“记事本”）。这个“记事本”通常看起来像一个[消息列表](./low_level.md#why-use-messages)。共享完整思考过程的好处是，它可以帮助其他智能体做出更好的决策，并提高整个系统的推理能力。缺点是，随着智能体的数量和复杂性的增长，“记事本”会迅速增长，可能需要额外的[内存管理](../how-tos/memory/add-memory.md)策略。

#### Sharing only final results (仅共享最终结果)

智能体可以拥有自己的私有“记事本”，并且只将[最终结果与其余智能体共享]。对于拥有众多智能体或复杂智能体的系统，这种方法可能效果更好。在这种情况下，您需要使用[不同的状态模式](#using-different-state-schemas)来定义智能体。

对于作为工具调用的智能体，Supervisor 根据工具模式确定输入。此外，LangGraph 允许[在运行时将状态传递](../how-tos/tool-calling.md#short-term-memory)给单个工具，因此从属智能体可以在需要时访问父状态。

#### Indicating agent name in messages (在消息中指示智能体名称)

指明特定 AI 消息来自哪个智能体可能很有帮助，尤其是在消息历史记录很长的情况下。一些 LLM 提供商（例如 OpenAI）支持向消息添加 `name` 参数 — 您可以使用它将智能体名称附加到消息中。如果不支持，您可以考虑将智能体名称手动注入到消息内容中，例如 `<agent>alice</agent><message>message from alice</message>`。

### Representing handoffs in message history (在消息历史中表示交接)

交接通常通过 LLM 调用专用的[交接工具](#handoffs-as-tools)来完成。这表示为传递给下一个智能体（LLM）的[AI 消息](https://python.langchain.com/docs/concepts/messages/#aimessage)（其中包含工具调用）。大多数 LLM 提供商不支持在**没有**相应工具消息的情况下接收带有工具调用的 AI 消息。

因此，您有两种选择：

1. 向消息列表中添加一个额外的[工具消息](https://python.langchain.com/docs/concepts/messages/#toolmessage)，例如，“成功转移到智能体 X”。
2. 删除带有工具调用的 AI 消息。

在实践中，我们发现大多数开发人员选择选项（1）。

### State management for subagents (子智能体的状态管理)

一种常见做法是让多个智能体在共享的消息列表上进行通信，但只[将它们的最终消息添加到列表中](#sharing-only-final-results)。这意味着任何中间消息（例如，工具调用）都不会保存在此列表中。

如果您确实希望保存这些消息，以便将来调用此特定子智能体时可以将其传回，该怎么办？

有两种高级方法可以实现这一点：

1. 将这些消息存储在共享的消息列表中，但在将列表传递给子智能体 LLM 之前对其进行过滤。例如，您可以选择过滤掉来自**其他**智能体的所有工具调用。
2. 为每个智能体存储单独的消息列表（例如 `alice_messages`），并在子智能体的图状态中。这将是它们对消息历史记录外观的“视图”。

### Using different state schemas (使用不同的状态模式)

智能体可能需要具有不同于其余智能体的状态模式。例如，搜索智能体可能只需要跟踪查询和检索到的文档。在 LangGraph 中实现此目的有两种方法：

- 使用单独的状态模式定义[子图](./subgraphs.md)智能体。如果子图和父图之间没有共享状态键（通道），则[添加输入/输出转换](../how-tos/subgraph.md#different-state-schemas)很重要，以便父图知道如何与子图通信。
- 使用具有[私有输入状态模式](../how-tos/graph-api.md/#pass-private-state-between-nodes)的智能体节点函数，该模式与整体图状态模式不同。这允许传递仅在执行特定智能体时需要的信息。