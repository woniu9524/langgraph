# 多代理系统

[Agent](./agentic_concepts.md#agent-architectures) 是_一个使用 LLM 来决定应用程序控制流的系统_。随着您开发这些系统，它们可能会随着时间的推移而变得更加复杂，从而更难管理和扩展。例如，您可能会遇到以下问题：

- 代理可用的工具过多，在决定下一个调用哪个工具时会做出错误的决策
- 上下文对于单个代理来说变得过于复杂，难以跟踪
- 系统中存在多个专业化领域的需求（例如，规划师、研究员、数学专家等）

为了解决这些问题，您可以考虑将应用程序分解为多个更小、独立的代理，并将它们组合成一个 **多代理系统**。这些独立的代理可以很简单，仅包含一个提示和一个 LLM 调用，也可以很复杂，如同一个 [ReAct](./agentic_concepts.md#tool-calling-agent) 代理（甚至更复杂！）。

使用多代理系统的主要优势包括：

- **模块化**：独立的代理可以更轻松地开发、测试和维护智能系统。
- **专业化**：您可以创建专注于特定领域的专家代理，这有助于提高整个系统的性能。
- **控制**：您可以显式控制代理之间的通信方式（而不是依赖函数调用）。

## 多代理架构

![](./img/multi_agent/architectures.png)

在多代理系统中连接代理有几种方法：

- **网络**：每个代理都可以与 [其他所有代理](../tutorials/multi_agent/multi-agent-collaboration.ipynb/) 进行通信。任何代理都可以决定接下来调用哪个其他代理。
- **Supervisor（监控器）**：每个代理与单个 [supervisor](../tutorials/multi_agent/agent_supervisor.md/) 代理通信。supervisor 代理负责决定接下来应调用哪个代理。
- **Supervisor（工具调用）**：这是 supervisor 架构的一个特例。可以将单个代理表示为工具。在这种情况下，supervisor 代理使用工具调用 LLM 来决定调用哪个代理工具以及传递给这些代理的参数。
- **分层**：您可以使用 [supervisor 的 supervisor](../tutorials/multi_agent/hierarchical_agent_teams.ipynb/) 来定义多代理系统。这是 supervisor 架构的泛化，并允许更复杂的控制流。
- **自定义多代理工作流**：每个代理仅与一部分代理通信。流程的某些部分是确定的，只有部分代理可以决定接下来调用哪些其他代理。

### 交接

在多代理架构中，代理可以表示为图节点。每个代理节点执行其步骤，然后决定是完成执行还是路由到另一个代理，包括可能路由到自身（例如，循环运行）。多代理交互中的一个常见模式是 **交接（handoffs）**，即一个代理将控制权 _交接_ 给另一个代理。交接允许您指定：

- **目标**：要导航到的目标代理（例如，要去的节点的名称）
- **负载（payload）**：[要传递给该代理的信息](#communication-and-state-management)（例如，状态更新）

要在 LangGraph 中实现交接，代理节点可以返回 [`Command`](./low_level.md#command) 对象，该对象允许您同时包含控制流和状态更新：

:::python

```python
def agent(state) -> Command[Literal["agent", "another_agent"]]:
    # 路由/停止的条件可以是任何东西，例如 LLM 工具调用 / 结构化输出等。
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    return Command(
        # 指定要调用的下一个代理
        goto=goto,
        # 更新图状态
        update={"my_state_key": "my_state_value"}
    )
```

:::

:::js

```typescript
graph.addNode((state) => {
    // 路由/停止的条件可以是任何东西，例如 LLM 工具调用 / 结构化输出等。
    const goto = getNextAgent(...); // 'agent' / 'another_agent'
    return new Command({
      // 指定要调用的下一个代理
      goto,
      // 更新图状态
      update: { myStateKey: "myStateValue" }
    });
})
```

:::

:::python
在更复杂的场景中，其中每个代理节点本身就是一个图（即，一个 [子图](./subgraphs.md)），并且一个代理子图中的节点可能想要导航到另一个代理。例如，如果您有两个代理 `alice` 和 `bob`（父图中的子图节点），并且 `alice` 需要导航到 `bob`，您可以在 `Command` 对象中将 `graph=Command.PARENT` 设置为：

```python
def some_node_inside_alice(state):
    return Command(
        goto="bob",
        update={"my_state_key": "my_state_value"},
        # 指定要导航到的图（默认为当前图）
        graph=Command.PARENT,
    )
```

:::

:::js
在更复杂的场景中，其中每个代理节点本身就是一个图（即，一个 [子图](./subgraphs.md)），并且一个代理子图中的节点可能想要导航到另一个代理。例如，如果您有两个代理 `alice` 和 `bob`（父图中的子图节点），并且 `alice` 需要导航到 `bob`，您可以在 `Command` 对象中将 `graph: Command.PARENT` 设置为：

```typescript
alice.addNode((state) => {
  return new Command({
    goto: "bob",
    update: { myStateKey: "myStateValue" },
    // 指定要导航到的图（默认为当前图）
    graph: Command.PARENT,
  });
});
```

:::

!!! note

    :::python

    如果您需要支持使用 `Command(graph=Command.PARENT)` 进行通信的子图的可视化，您需要将它们包装在一个带有 `Command` 注释的节点函数中：
    而不是这样做：

    ```python
    builder.add_node(alice)
    ```

    您需要这样做：

    ```python
    def call_alice(state) -> Command[Literal["bob"]]:
        return alice.invoke(state)

    builder.add_node("alice", call_alice)
    ```

    :::

    :::js
    如果您需要支持使用 / `Command({ graph: Command.PARENT })` 进行通信的子图的可视化，您需要将它们包装在一个带有 `Command` 注释的节点函数中：

    而不是这样做：

    ```typescript
    builder.addNode("alice", alice);
    ```

    您需要这样做：

    ```typescript
    builder.addNode("alice", (state) => alice.invoke(state), { ends: ["bob"] });
    ```

    :::

#### 将交接作为工具

最常见的代理类型之一是 [工具调用代理](../agents/overview.md)。对于这些类型的代理，常见的模式是将交接包装在工具调用中：

:::python

```python
from langchain_core.tools import tool

@tool
def transfer_to_bob():
    """Transfer to bob."""
    return Command(
        # 要去的代理（节点）的名称
        goto="bob",
        # 要发送给代理的数据
        update={"my_state_key": "my_state_value"},
        # 告知 LangGraph 我们需要导航到
        # 父图中的代理节点
        graph=Command.PARENT,
    )
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { Command } from "@langchain/langgraph";
import { z } from "zod";

const transferToBob = tool(
  async () => {
    return new Command({
      // 要去的代理（节点）的名称
      goto: "bob",
      // 要发送给代理的数据
      update: { myStateKey: "myStateValue" },
      // 告知 LangGraph 我们需要导航到
      // 父图中的代理节点
      graph: Command.PARENT,
    });
  },
  {
    name: "transfer_to_bob",
    description: "Transfer to bob.",
    schema: z.object({}),
  }
);
```

:::

这是从工具更新图状态的一个特例，除了状态更新之外，还包括了控制流。

!!! important

      :::python
      如果您想使用返回 `Command` 的工具，您可以使用预构建的 @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] 组件，否则请实现您自己的逻辑：

      ```python
      def call_tools(state):
          ...
          commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
          return commands
      ```
      :::

      :::js
      如果您想使用返回 `Command` 的工具，您可以使用预构建的 @[`createReactAgent`][create_react_agent] / @[ToolNode] 组件，否则请实现您自己的逻辑：

      ```typescript
      graph.addNode("call_tools", async (state) => {
        // ... tool execution logic
        const commands = toolCalls.map((toolCall) =>
          toolsByName[toolCall.name].invoke(toolCall)
        );
        return commands;
      });
      ```
      :::

现在，让我们仔细看看不同的多代理架构。

### 网络

在此架构中，代理定义为图节点。每个代理都可以与所有其他代理通信（多对多连接），并可以决定下一个调用哪个代理。这种架构适用于没有清晰的代理层级结构或没有特定调用顺序的问题。

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def agent_1(state: MessagesState) -> Command[Literal["agent_2", "agent_3", END]]:
    # 您可以将状态的相关部分传递给 LLM（例如，state["messages"]）
    # 来决定接下来调用哪个代理。一种常见的模式是使用结构化输出调用模型
    # （例如，强制它返回带有“next_agent”字段的输出）。
    response = model.invoke(...)
    # 根据 LLM 的决定路由到其中一个代理或退出
    # 如果 LLM 返回 "__end__"，则图将完成执行
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

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { Command } from "@langchain/langgraph";
import { z } from "zod";

const model = new ChatOpenAI();

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
  // 来决定接下来调用哪个代理。一种常见的模式是使用结构化输出调用模型
  // （例如，强制它返回带有“next_agent”字段的输出）。
  const response = await model.invoke(...);
  // 根据 LLM 的决定路由到其中一个代理或退出
  // 如果 LLM 返回 "__end__"，则图将完成执行
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const agent3 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
  return new Command({
    goto: response.nextAgent,
    update: { messages: [response.content] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("agent1", agent1, {
    ends: ["agent2", "agent3", END]
  })
  .addNode("agent2", agent2, {
    ends: ["agent1", "agent3", END]
  })
  .addNode("agent3", agent3, {
    ends: ["agent1", "agent2", END]
  })
  .addEdge(START, "agent1");

const network = builder.compile();
```

:::

### Supervisor（监控器）

在此架构中，我们将代理定义为节点，并添加一个 supervisor 节点（LLM）来决定应调用哪个代理节点。我们使用 [`Command`](./low_level.md#command) 根据 supervisor 的决定将执行路由到适当的代理节点。此架构也很适合并行运行多个代理或使用 [map-reduce](../how-tos/graph-api.md#map-reduce-and-the-send-api) 模式。

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.types import Command
from langgraph.graph import StateGraph, MessagesState, START, END

model = ChatOpenAI()

def supervisor(state: MessagesState) -> Command[Literal["agent_1", "agent_2", END]]:
    # 您可以将状态的相关部分传递给 LLM（例如，state["messages"]）
    # 来决定接下来调用哪个代理。一种常见的模式是使用结构化输出调用模型
    # （例如，强制它返回带有“next_agent”字段的输出）。
    response = model.invoke(...)
    # 根据 supervisor 的决定路由到其中一个代理或退出
    # 如果 supervisor 返回 "__end__"，则图将完成执行
    return Command(goto=response["next_agent"])

def agent_1(state: MessagesState) -> Command[Literal["supervisor"]]:
    # 您可以将状态的相关部分传递给 LLM（例如，state["messages"]）
    # 并添加任何其他逻辑（不同的模型、自定义提示、结构化输出等）。
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

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
  // 来决定接下来调用哪个代理。一种常见的模式是使用结构化输出调用模型
  // （例如，强制它返回带有“next_agent”字段的输出）。
  const response = await model.invoke(...);
  // 根据 supervisor 的决定路由到其中一个代理或退出
  // 如果 supervisor 返回 "__end__"，则图将完成执行
  return new Command({ goto: response.nextAgent });
};

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
  // 并添加任何其他逻辑（不同的模型、自定义提示、结构化输出等）。
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("supervisor", supervisor, {
    ends: ["agent1", "agent2", END]
  })
  .addNode("agent1", agent1, {
    ends: ["supervisor"]
  })
  .addNode("agent2", agent2, {
    ends: ["supervisor"]
  })
  .addEdge(START, "supervisor");

const supervisorGraph = builder.compile();
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
  // 来决定接下来调用哪个代理。一种常见的模式是使用结构化输出调用模型
  // （例如，强制它返回带有“next_agent”字段的输出）。
  const response = await model.invoke(...);
  // 根据 supervisor 的决定路由到其中一个代理或退出
  // 如果 supervisor 返回 "__end__"，则图将完成执行
  return new Command({ goto: response.nextAgent });
};

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
  // 并添加任何其他逻辑（不同的模型、自定义提示、结构化输出等）。
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "supervisor",
    update: { messages: [response] },
  });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("supervisor", supervisor, {
    ends: ["agent1", "agent2", END]
  })
  .addNode("agent1", agent1, {
    ends: ["supervisor"]
  })
  .addNode("agent2", agent2, {
    ends: ["supervisor"]
  })
  .addEdge(START, "supervisor");

const supervisorGraph = builder.compile();
```

:::

查看这个 [教程](../tutorials/multi_agent/agent_supervisor.md) 来了解 supervisor 多代理架构的示例。

### Supervisor（工具调用）

在此 [supervisor](#supervisor) 架构的变体中，我们定义了一个 supervisor [代理](./agentic_concepts.md#agent-architectures)，它负责调用子代理。子代理以工具的形式暴露给 supervisor，supervisor 代理决定了下一个要调用的工具。supervisor 代理遵循标准的 [实现](./agentic_concepts.md#tool-calling-agent)，即一个 LLM 在一个 while 循环中运行，调用工具直到它决定停止。

:::python

```python
from typing import Annotated
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import InjectedState, create_react_agent

model = ChatOpenAI()

# 这是将作为工具调用的代理函数
# 请注意，您可以通过 InjectedState 注释将状态传递给工具
def agent_1(state: Annotated[dict, InjectedState]):
    # 您可以将状态的相关部分传递给 LLM（例如，state["messages"]）
    # 并添加任何其他逻辑（不同的模型、自定义提示、结构化输出等）。
    response = model.invoke(...)
    # 以字符串形式返回 LLM 响应（预期的工具响应格式）
    # 这将由预构建的 create_react_agent（supervisor）自动转换为 ToolMessage
    return response.content

def agent_2(state: Annotated[dict, InjectedState]):
    response = model.invoke(...)
    return response.content

tools = [agent_1, agent_2]
# 构建带有工具调用的 supervisor 的最简单方法是使用预构建的 ReAct 代理图
# 它由一个工具调用 LLM 节点（即 supervisor）和一个工具执行节点组成
supervisor = create_react_agent(model, tools)
```

:::

:::js

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const model = new ChatOpenAI();

// 这是将作为工具调用的代理函数
// 请注意，您可以通过 config 参数将状态传递给工具
const agent1 = tool(
  async (_, config) => {
    const state = config.configurable?.state;
    // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
    // 并添加任何其他逻辑（不同的模型、自定义提示、结构化输出等）。
    const response = await model.invoke(...);
    // 以字符串形式返回 LLM 响应（预期的工具响应格式）
    // 这将由预构建的 createReactAgent（supervisor）自动转换为 ToolMessage
    return response.content;
  },
  {
    name: "agent1",
    description: "Agent 1 description",
    schema: z.object({}),
  }
);

const agent2 = tool(
  async (_, config) => {
    const state = config.configurable?.state;
    const response = await model.invoke(...);
    return response.content;
  },
  {
    name: "agent2",
    description: "Agent 2 description",
    schema: z.object({}),
  }
);

const tools = [agent1, agent2];
// 构建带有工具调用的 supervisor 的最简单方法是使用预构建的 ReAct 代理图
// 它由一个工具调用 LLM 节点（即 supervisor）和一个工具执行节点组成
const supervisor = createReactAgent({ llm: model, tools });
```

:::

### 分层

当您向系统中添加更多代理时，supervisor 可能很难管理所有这些代理。supervisor 可能会在决定下一个调用哪个代理时做出糟糕的决策，或者上下文可能会变得过于复杂，以至于单个 supervisor 无法跟踪。换句话说，您会遇到最初促使采用多代理架构的相同问题。

为了解决这个问题，您可以 _分层_ 设计您的系统。例如，您可以创建由单个 supervisor 管理的单独的专业化代理团队，以及一个负责管理这些团队的顶层 supervisor。

:::python

```python
from typing import Literal
from langchain_openai import ChatOpenAI
from langgraph.graph import StateGraph, MessagesState, START, END
from langgraph.types import Command
model = ChatOpenAI()

# 定义团队 1（与单个 supervisor 示例相同）

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

# 定义团队 2（与单个 supervisor 示例相同）
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


# 定义顶层 supervisor

builder = StateGraph(MessagesState)
def top_level_supervisor(state: MessagesState) -> Command[Literal["team_1_graph", "team_2_graph", END]]:
    # 您可以将状态的相关部分传递给 LLM（例如，state["messages"]）
    # 来决定接下来调用哪个团队。一种常见的模式是调用模型
    # 并带有结构化输出（例如，强制它返回一个带有“next_team”字段的输出）。
    response = model.invoke(...)
    # 根据 supervisor 的决定路由到其中一个团队或退出
    # 如果 supervisor 返回 "__end__"，则图将完成执行
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

:::

:::js

```typescript
import { StateGraph, MessagesZodState, Command, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

// 定义团队 1（与单个 supervisor 示例相同）

const team1Supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({ goto: response.nextAgent });
};

const team1Agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "team1Supervisor",
    update: { messages: [response] }
  });
};

const team1Agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return new Command({
    goto: "team1Supervisor",
    update: { messages: [response] }
  });
};

const team1Builder = new StateGraph(MessagesZodState)
  .addNode("team1Supervisor", team1Supervisor, {
    ends: ["team1Agent1", "team1Agent2", END]
  })
  .addNode("team1Agent1", team1Agent1, {
    ends: ["team1Supervisor"]
  })
  .addNode("team1Agent2", team1Agent2, {
    ends: ["team1Supervisor"]
  })
  .addEdge(START, "team1Supervisor");
const team1Graph = team1Builder.compile();

// 定义团队 2（与单个 supervisor 示例相同）
const team2Supervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  // ...
};

const team2Builder = new StateGraph(MessagesZodState);
// ... 构建 team2Graph
const team2Graph = team2Builder.compile();

// 定义顶层 supervisor

const topLevelSupervisor = async (state: z.infer<typeof MessagesZodState>) => {
  // 您可以将状态的相关部分传递给 LLM（例如，state.messages）
  // 来决定接下来调用哪个团队。一种常见的模式是调用模型
  // 并带有结构化输出（例如，强制它返回一个带有“next_team”字段的输出）。
  const response = await model.invoke(...);
  // 根据 supervisor 的决定路由到其中一个团队或退出
  // 如果 supervisor 返回 "__end__"，则图将完成执行
  return new Command({ goto: response.nextTeam });
};

const builder = new StateGraph(MessagesZodState)
  .addNode("topLevelSupervisor", topLevelSupervisor, {
    ends: ["team1Graph", "team2Graph", END]
  })
  .addNode("team1Graph", team1Graph)
  .addNode("team2Graph", team2Graph)
  .addEdge(START, "topLevelSupervisor")
  .addEdge("team1Graph", "topLevelSupervisor")
  .addEdge("team2Graph", "topLevelSupervisor");

const graph = builder.compile();
```

:::

### 自定义多代理工作流

在此架构中，我们将单个代理添加为图节点，并预先定义代理的调用顺序，形成自定义工作流。在 LangGraph 中，工作流可以有两种方式定义：

- **显式控制流（普通边）**：LangGraph 允许您通过 [普通图边](./low_level.md#normal-edges) 显式定义应用程序的控制流（即代理通信的顺序）。这是此架构中最确定的变体 — 我们始终提前知道将调用哪个代理。

- **动态控制流（Command）**：在 LangGraph 中，您可以允许 LLM 决定应用程序控制流的部分。这可以通过使用 [`Command`](./low_level.md#command) 来实现。一个特例是 [supervisor 工具调用](#supervisor-tool-calling) 架构。在这种情况下，驱动 supervisor 代理的工具调用 LLM 将决定工具（代理）的调用顺序。

:::python

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
# 显式定义流程
builder.add_edge(START, "agent_1")
builder.add_edge("agent_1", "agent_2")
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const model = new ChatOpenAI();

const agent1 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return { messages: [response] };
};

const agent2 = async (state: z.infer<typeof MessagesZodState>) => {
  const response = await model.invoke(...);
  return { messages: [response] };
};

const builder = new StateGraph(MessagesZodState)
  .addNode("agent1", agent1)
  .addNode("agent2", agent2)
  // 显式定义流程
  .addEdge(START, "agent1")
  .addEdge("agent1", "agent2");
```

:::

## 通信和状态管理

构建多代理系统时最重要的事情是弄清楚代理如何通信。

代理之间的一种常见、通用的通信方式是通过消息列表。这引出了以下问题：

- 代理是 [**通过交接还是通过工具调用**](#handoffs-vs-tool-calls) 进行通信？
- [**哪些消息会从一个代理传递给下一个代理**](#message-passing-between-agents)？
- [**如何体现交接在消息列表中的表示**](#representing-handoffs-in-message-history)？
- 如何 [**管理子代理的状态**](#state-management-for-subagents)？

此外，如果您要处理更复杂的代理或希望将单个代理的状态与多代理系统状态分开，您可能需要使用 [**不同的状态模式**](#using-different-state-schemas)。

### 交接与工具调用

在代理之间传递的 "payload" 是什么？在上面讨论的大多数架构中，代理通过 [交接](#handoffs) 进行通信，并将 [图状态](./low_level.md#state) 作为交接 payload 的一部分进行传递。具体来说，代理在图状态中传递消息列表。在 [带工具调用的 supervisor](#supervisor-tool-calling) 的情况下，payload 是工具调用的参数。

![](./img/multi_agent/request.png)

### 代理之间的消息传递

代理之间最常见的通信方式是通过共享状态通道，通常是一个消息列表。这假设至少有一个通道（键），所有代理共享该通道（例如 `messages`）。通过共享消息列表进行通信时，还有一个额外的考虑因素：代理应该 [共享其思考过程的完整历史记录](#sharing-full-thought-process) 还是仅 [共享最终结果](#sharing-only-final-results)？

![](./img/multi_agent/response.png)

#### 共享完整的思考过程

代理可以将其思考过程的 **完整历史记录**（即“草稿板”）与所有其他代理 **共享**。这个“草稿板”通常看起来像一个 [消息列表](./low_level.md#why-use-messages)。共享完整思考过程的好处是，它可以帮助其他代理做出更好的决策，并提高整个系统的推理能力。缺点是，随着代理数量的增加和复杂性的提高，“草稿板”会迅速增长，并可能需要额外的 [内存管理](../how-tos/memory/add-memory.md) 策略。

#### 只共享最终结果

代理可以拥有自己的私有“草稿板”，并仅将 **最终结果** 与其他代理 **共享**。这种方法可能更适用于拥有许多代理或更复杂代理的系统。在这种情况下，您需要定义具有 [不同状态模式](#using-different-state-schemas) 的代理。

作为工具调用的代理，supervisor 会根据工具模式确定输入。此外，LangGraph 允许在运行时 [将状态传递给单个工具](../how-tos/tool-calling.md#short-term-memory) ，因此下级代理可以根据需要访问父状态。

#### 在消息中指示代理名称

指示特定 AI 消息来自哪个代理可能很有帮助，尤其是在消息历史记录很长的情况下。一些 LLM 提供商（例如 OpenAI）支持向消息添加 `name` 参数 — 您可以使用它将代理名称附加到消息中。如果不支持，您可以考虑手动将代理名称注入消息内容，例如 `<agent>alice</agent><message>message from alice</message>`。

### 在消息历史记录中表示交接

:::python
交接通常是通过 LLM 调用专用 [交接工具](#handoffs-as-tools) 来完成的。这表示为传递给下一个代理（LLM）的带有工具调用的 [AI 消息](https://python.langchain.com/docs/concepts/messages/#aimessage)。大多数 LLM 提供商不支持接收带有工具调用的 AI 消息，**但没有** 相应的工具消息。
:::

:::js
交接通常是通过 LLM 调用专用 [交接工具](#handoffs-as-tools) 来完成的。这表示为传递给下一个代理（LLM）的带有工具调用的 [AI 消息](https://js.langchain.com/docs/concepts/messages/#aimessage)。大多数 LLM 提供商不支持接收带有工具调用的 AI 消息，**但没有** 相应的工具消息。
:::

因此，您有两个选择：

:::python

1. 向消息列表中添加一个额外的 [工具消息](https://python.langchain.com/docs/concepts/messages/#toolmessage) ，例如，“成功转移到代理 X”。
2. 删除带有工具调用的 AI 消息。
   :::

:::js

1. 向消息列表中添加一个额外的 [工具消息](https://js.langchain.com/docs/concepts/messages/#toolmessage) ，例如，“成功转移到代理 X”。
2. 删除带有工具调用的 AI 消息。
:::

在实践中，我们发现大多数开发人员会选择选项 (1)。

### 子代理的状态管理

一个常见的做法是让多个代理在共享的消息列表上进行通信，但只 [将它们的最终消息添加到列表中](#sharing-only-final-results)。这意味着任何中间消息（例如，工具调用）都不会保存在此列表中。

如果您确实想保存这些消息，以便以后调用此特定子代理时可以再次传递它们，该怎么办？

有两种高级方法可以实现这一点：

:::python

1. 将这些消息存储在共享的消息列表中，但先过滤该列表，然后再将其传递给子代理 LLM。例如，您可以选择过滤掉 **其他** 代理的所有工具调用。
2. 为每个代理（例如 `alice_messages`）在子代理的图状态中存储单独的消息列表。这将是它们对消息历史记录外观的“视图”。
:::

:::js

1. 将这些消息存储在共享的消息列表中，但先过滤该列表，然后再将其传递给子代理 LLM。例如，您可以选择过滤掉 **其他** 代理的所有工具调用。
2. 为每个代理（例如 `aliceMessages`）在子代理的图状态中存储单独的消息列表。这将是它们对消息历史记录外观的“视图”。
:::

### 使用不同的状态模式

代理可能需要与其余代理具有不同的状态模式。例如，搜索代理可能只需要跟踪查询和检索到的文档。在 LangGraph 中实现这一点有两种方法：

- 使用单独的状态模式定义 [子图](./subgraphs.md) 代理。如果子图和父图之间没有共享状态键（通道），则务必 [添加输入/输出转换](../how-tos/subgraph.md#different-state-schemas) ，以便父图知道如何与子图通信。
- 使用 [私有输入状态模式](../how-tos/graph-api.md#pass-private-state-between-nodes) 定义代理节点函数，该模式不同于整体图状态模式。这允许传递仅用于执行该特定代理的信息。