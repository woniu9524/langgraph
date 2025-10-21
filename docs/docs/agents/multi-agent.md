---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 多智能体 (Multi-agent)

当一个智能体需要专注于多个领域或管理大量工具时，它可能会面临挑战。为了解决这个问题，你可以将你的智能体分解成更小、独立的智能体，并将它们组合成一个[多智能体系统](../concepts/multi_agent.md)。

在多智能体系统中，智能体之间需要进行通信。它们通过[交接 (handoffs)](#handoffs) 来实现——这是一种原始操作，用于描述将控制权交给哪个智能体以及要发送给该智能体的载荷 (payload)。

最流行的两种多智能体架构是：

- [Supervisor](#supervisor) — 由一个中心性的 supervisor 智能体协调各个独立的智能体。Supervisor 控制所有通信流程和任务委派，并根据当前上下文和任务需求来决定调用哪个智能体。
- [Swarm](#swarm) — 智能体根据它们的专业化能力动态地将控制权交接给彼此。系统会记住上次活动的智能体，确保在后续交互中，会话可以与该智能体恢复。

## Supervisor (主管)

![Supervisor](./assets/supervisor.png)

使用 [`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-py) 库来创建一个 supervisor 多智能体系统：

```bash
pip install langgraph-supervisor
```

```python
from langchain_openai import ChatOpenAI
from langgraph.prebuilt import create_react_agent
# highlight-next-line
from langgraph_supervisor import create_supervisor

def book_hotel(hotel_name: str):
    """预订酒店"""
    return f"成功预订了 {hotel_name} 的住宿。."

def book_flight(from_airport: str, to_airport: str):
    """预订航班"""
    return f"成功预订了从 {from_airport} 到 {to_airport} 的航班。"

flight_assistant = create_react_agent(
    model="openai:gpt-4o",
    tools=[book_flight],
    prompt="你是一名航班预订助手",
    # highlight-next-line
    name="flight_assistant"
)

hotel_assistant = create_react_agent(
    model="openai:gpt-4o",
    tools=[book_hotel],
    prompt="你是一名酒店预订助手",
    # highlight-next-line
    name="hotel_assistant"
)

# highlight-next-line
supervisor = create_supervisor(
    agents=[flight_assistant, hotel_assistant],
    model=ChatOpenAI(model="gpt-4o"),
    prompt=(
        "你管理着一个酒店预订助手和一个"
        "航班预订助手。请将工作分配给它们。"
    )
).compile()

for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "预订从 BOS 到 JFK 的航班以及 McKittrick Hotel 的住宿"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

使用 [`@langchain/langgraph-supervisor`](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-supervisor) 库来创建一个 supervisor 多智能体系统：

```bash
npm install @langchain/langgraph-supervisor
```

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
// highlight-next-line
import { createSupervisor } from "langgraph-supervisor";

function bookHotel(hotelName: string) {
  /**预订酒店*/
  return `成功预订了 ${hotelName} 的住宿。`;
}

function bookFlight(fromAirport: string, toAirport: string) {
  /**预订航班*/
  return `成功预订了从 ${fromAirport} 到 ${toAirport} 的航班。`;
}

const flightAssistant = createReactAgent({
  llm: "openai:gpt-4o",
  tools: [bookFlight],
  stateModifier: "你是一名航班预订助手",
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: "openai:gpt-4o",
  tools: [bookHotel],
  stateModifier: "你是一名酒店预订助手",
  // highlight-next-line
  name: "hotel_assistant",
});

// highlight-next-line
const supervisor = createSupervisor({
  agents: [flightAssistant, hotelAssistant],
  llm: new ChatOpenAI({ model: "gpt-4o" }),
  systemPrompt:
    "你管理着一个酒店预订助手和一个 " +
    "航班预订助手。请将工作分配给它们。",
});

for await (const chunk of supervisor.stream({
  messages: [
    {
      role: "user",
      content: "预订从 BOS 到 JFK 的航班以及 McKittrick Hotel 的住宿",
    },
  ],
})) {
  console.log(chunk);
  console.log("\n");
}
```

## Swarm (蜂群)

![Swarm](./assets/swarm.png)

使用 [`langgraph-swarm`](https://github.com/langchain-ai/langgraph-swarm-py) 库来创建一个 swarm 多智能体系统：

```bash
pip install langgraph-swarm
```

```python
from langgraph.prebuilt import create_react_agent
# highlight-next-line
from langgraph_swarm import create_swarm, create_handoff_tool

transfer_to_hotel_assistant = create_handoff_tool(
    agent_name="hotel_assistant",
    description="将用户转接至酒店预订助手。",
)
transfer_to_flight_assistant = create_handoff_tool(
    agent_name="flight_assistant",
    description="将用户转接至航班预订助手。",
)

flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_flight, transfer_to_hotel_assistant],
    prompt="你是一名航班预订助手",
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_hotel, transfer_to_flight_assistant],
    prompt="你是一名酒店预订助手",
    # highlight-next-line
    name="hotel_assistant"
)

# highlight-next-line
swarm = create_swarm(
    agents=[flight_assistant, hotel_assistant],
    default_active_agent="flight_assistant"
).compile()

for chunk in swarm.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "预订从 BOS 到 JFK 的航班以及 McKittrick Hotel 的住宿"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

使用 [`@langchain/langgraph-swarm`](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-swarm) 库来创建一个 swarm 多智能体系统：

```bash
npm install @langchain/langgraph-swarm
```

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
// highlight-next-line
import { createSwarm, createHandoffTool } from "@langchain/langgraph-swarm";

const transferToHotelAssistant = createHandoffTool({
  agentName: "hotel_assistant",
  description: "Transfer user to the hotel-booking assistant.",
});

const transferToFlightAssistant = createHandoffTool({
  agentName: "flight_assistant",
  description: "Transfer user to the flight-booking assistant.",
});

const flightAssistant = createReactAgent({
  llm: "anthropic:claude-3-5-sonnet-latest",
  // highlight-next-line
  tools: [bookFlight, transferToHotelAssistant],
  stateModifier: "You are a flight booking assistant",
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: "anthropic:claude-3-5-sonnet-latest",
  // highlight-next-line
  tools: [bookHotel, transferToFlightAssistant],
  stateModifier: "You are a hotel booking assistant",
  // highlight-next-line
  name: "hotel_assistant",
});

// highlight-next-line
const swarm = createSwarm({
  agents: [flightAssistant, hotelAssistant],
  defaultActiveAgent: "flight_assistant",
});

for await (const chunk of swarm.stream({
  messages: [
    {
      role: "user",
      content: "book a flight from BOS to JFK and a stay at McKittrick Hotel",
    },
  ],
})) {
  console.log(chunk);
  console.log("\n");
}
```

## Handoffs (交接)

多智能体交互中的一个常见模式是**交接 (handoffs)**，即一个智能体将控制权“交接”给另一个智能体。交接允许你指定：

- **destination (目标)**: 要导航到的目标智能体
- **payload (载荷)**: 要传递给该智能体的信息

这既被 `langgraph-supervisor`（supervisor 将控制权交接给各个智能体）和 `langgraph-swarm`（单个智能体可以将控制权交接给其他智能体）使用。

要使用 `create_react_agent` 实现交接，你需要：

1.  创建一个可以向不同智能体传输控制权的特殊工具。

    ```python
    def transfer_to_bob():
        """转接给 Bob。"""
        return Command(
            # 要前往的智能体 (节点) 的名称
            # highlight-next-line
            goto="bob",
            # 要发送给智能体的数据
            # highlight-next-line
            update={"messages": [...]},
            # 指示 LangGraph 需要导航到
            # 父图中的智能体节点
            # highlight-next-line
            graph=Command.PARENT,
        )
    ```

2.  创建可以访问交接工具的各个智能体：

    ```python
    flight_assistant = create_react_agent(
        ..., tools=[book_flight, transfer_to_hotel_assistant]
    )
    hotel_assistant = create_react_agent(
        ..., tools=[book_hotel, transfer_to_flight_assistant]
    )
    ```

3.  定义一个包含各个智能体作为节点的父图：

    ```python
    from langgraph.graph import StateGraph, MessagesState
    multi_agent_graph = (
        StateGraph(MessagesState)
        .add_node(flight_assistant)
        .add_node(hotel_assistant)
        ...
    )
    ```

这既被 `@langchain/langgraph-supervisor`（supervisor 将控制权交接给各个智能体）和 `@langchain/langgraph-swarm`（单个智能体可以将控制权交接给其他智能体）使用。

要使用 `createReactAgent` 实现交接，你需要：

1.  创建一个可以向不同智能体传输控制权的特殊工具。

    ```typescript
    function transferToBob() {
      /**转接给 Bob。*/
      return new Command({
        // 要前往的智能体 (节点) 的名称
        // highlight-next-line
        goto: "bob",
        // 要发送给智能体的数据
        // highlight-next-line
        update: { messages: [...] },
        // 指示 LangGraph 需要导航到
        // 父图中的智能体节点
        // highlight-next-line
        graph: Command.PARENT,
      });
    }
    ```

2.  创建可以访问交接工具的各个智能体：

    ```typescript
    const flightAssistant = createReactAgent({
      ..., tools: [bookFlight, transferToHotelAssistant]
    });
    const hotelAssistant = createReactAgent({
      ..., tools: [bookHotel, transferToFlightAssistant]
    });
    ```

3.  定义一个包含各个智能体作为节点的父图：

    ```typescript
    import { StateGraph, MessagesZodState } from "@langchain/langgraph";
    const multiAgentGraph = new StateGraph(MessagesZodState)
      .addNode("flight_assistant", flightAssistant)
      .addNode("hotel_assistant", hotelAssistant)
      // ...
    ```

将这些组合在一起，以下是实现一个简单的多智能体系统的方法，该系统包含两个智能体——一个航班预订助手和一个酒店预订助手：

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import create_react_agent, InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Transfer to {agent_name}"

    @tool(name, description=description)
    def handoff_tool(
        # highlight-next-line
        state: Annotated[MessagesState, InjectedState], # (1)!
        # highlight-next-line
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        return Command(  # (2)!
            # highlight-next-line
            goto=agent_name,  # (3)!
            # highlight-next-line
            update={"messages": state["messages"] + [tool_message]},  # (4)!
            # highlight-next-line
            graph=Command.PARENT,  # (5)!
        )
    return handoff_tool

# 交接
transfer_to_hotel_assistant = create_handoff_tool(
    agent_name="hotel_assistant",
    description="将用户转接至酒店预订助手。",
)
transfer_to_flight_assistant = create_handoff_tool(
    agent_name="flight_assistant",
    description="将用户转接至航班预订助手。",
)

# 简单的智能体工具
def book_hotel(hotel_name: str):
    """预订酒店"""
    return f"成功预订了 {hotel_name} 的住宿。"

def book_flight(from_airport: str, to_airport: str):
    """预订航班"""
    return f"成功预订了从 {from_airport} 到 {to_airport} 的航班。"

# 定义智能体
flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_flight, transfer_to_hotel_assistant],
    prompt="你是一名航班预订助手",
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[book_hotel, transfer_to_flight_assistant],
    prompt="你是一名酒店预订助手",
    # highlight-next-line
    name="hotel_assistant"
)

# 定义多智能体图
multi_agent_graph = (
    StateGraph(MessagesState)
    .add_node(flight_assistant)
    .add_node(hotel_assistant)
    .add_edge(START, "flight_assistant")
    .compile()
)

# 运行多智能体图
for chunk in multi_agent_graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "预订从 BOS 到 JFK 的航班以及 McKittrick Hotel 的住宿"
            }
        ]
    }
):
    print(chunk)
    print("\n")
```

1.  访问智能体的状态
2.  `Command` 原始操作允许将状态更新和节点转换作为单个操作进行指定，这对于实现交接很有用。
3.  要交接到的智能体或节点的名称。
4.  获取智能体的消息，并将它们作为交接的一部分**添加**到父级的**状态**中。下一个智能体将看到父级状态。
5.  指示 LangGraph 需要导航到**父级**多智能体图中的智能体节点。

```typescript
import { tool } from "@langchain/core/tools";
import { ChatAnthropic } from "@langchain/anthropic";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import {
  StateGraph,
  START,
  MessagesZodState,
  Command,
} from "@langchain/langgraph";
import { z } from "zod";

function createHandoffTool({
  agentName,
  description,
}: {
  agentName: string;
  description?: string;
}) {
  const name = `transfer_to_${agentName}`;
  const toolDescription = description || `Transfer to ${agentName}`;

  return tool(
    async (_, config) => {
      const toolMessage = {
        role: "tool" as const,
        content: `Successfully transferred to ${agentName}`,
        name: name,
        tool_call_id: config.toolCall?.id!,
      };
      return new Command({
        // (2)!
        // highlight-next-line
        goto: agentName, // (3)!
        // highlight-next-line
        update: { messages: [toolMessage] }, // (4)!
        // highlight-next-line
        graph: Command.PARENT, // (5)!
      });
    },
    {
      name,
      description: toolDescription,
      schema: z.object({}),
    }
  );
}

// Handoffs (交接)
const transferToHotelAssistant = createHandoffTool({
  agentName: "hotel_assistant",
  description: "将用户转接至酒店预订助手。",
});

const transferToFlightAssistant = createHandoffTool({
  agentName: "flight_assistant",
  description: "将用户转接至航班预订助手。",
});

// Simple agent tools (简单的智能体工具)
const bookHotel = tool(
  async ({ hotelName }) => {
    /**预订酒店*/
    return `成功预订了 ${hotelName} 的住宿。`;
  },
  {
    name: "book_hotel",
    description: "预订酒店",
    schema: z.object({
      hotelName: z.string().describe("要预订的酒店名称"),
    }),
  }
);

const bookFlight = tool(
  async ({ fromAirport, toAirport }) => {
    /**预订航班*/
    return `成功预订了从 ${fromAirport} 到 ${toAirport} 的航班。`;
  },
  {
    name: "book_flight",
    description: "预订航班",
    schema: z.object({
      fromAirport: z.string().describe("出发机场代码"),
      toAirport: z.string().describe("到达机场代码"),
    }),
  }
);

// Define agents (定义智能体)
const flightAssistant = createReactAgent({
  llm: new ChatAnthropic({ model: "anthropic:claude-3-5-sonnet-latest" }),
  // highlight-next-line
  tools: [bookFlight, transferToHotelAssistant],
  stateModifier: "你是一名航班预订助手",
  // highlight-next-line
  name: "flight_assistant",
});

const hotelAssistant = createReactAgent({
  llm: new ChatAnthropic({ model: "anthropic:claude-3-5-sonnet-latest" }),
  // highlight-next-line
  tools: [bookHotel, transferToFlightAssistant],
  stateModifier: "你是一名酒店预订助手",
  // highlight-next-line
  name: "hotel_assistant",
});

// Define multi-agent graph (定义多智能体图)
const multiAgentGraph = new StateGraph(MessagesZodState)
  .addNode("flight_assistant", flightAssistant)
  .addNode("hotel_assistant", hotelAssistant)
  .addEdge(START, "flight_assistant")
  .compile();

// Run the multi-agent graph (运行多智能体图)
for await (const chunk of multiAgentGraph.stream({
  messages: [
    {
      role: "user",
      content: "预订从 BOS 到 JFK 的航班以及 McKittrick Hotel 的住宿",
    },
  ],
})) {
  console.log(chunk);
  console.log("\n");
}
```

1.  访问智能体的状态
2.  `Command` 原始操作允许将状态更新和节点转换作为单个操作进行指定，这对于实现交接很有用。
3.  要交接到的智能体或节点的名称。
4.  获取智能体的消息，并将它们作为交接的一部分**添加**到父级的**状态**中。下一个智能体将看到父级状态。
5.  指示 LangGraph 需要导航到**父级**多智能体图中的智能体节点。

!!! Note

    此交接实现假定：

    - 每个智能体接收到的输入是多智能体系统中整体的消息历史（跨所有智能体）。
    - 每个智能体将内部消息历史输出到多智能体系统的整体消息历史中。

[*]

    请查阅 LangGraph [supervisor](https://github.com/langchain-ai/langgraph-supervisor-py#customizing-handoff-tools) 和 [swarm](https://github.com/langchain-ai/langgraph-swarm-py#customizing-handoff-tools) 文档，了解如何自定义交接工具。
[*]

    请查阅 LangGraph [supervisor](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-supervisor#customizing-handoff-tools) 和 [swarm](https://github.com/langchain-ai/langgraphjs/tree/main/libs/langgraph-swarm#customizing-handoff-tools) 文档，了解如何自定义交接工具。