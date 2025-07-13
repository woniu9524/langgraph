# 构建多代理系统

单个代理在需要专精于多个领域或管理许多工具时可能会遇到困难。为解决此问题，您可以将代理分解为更小、独立的代理，并将它们组合成一个[多代理系统](../concepts/multi_agent.md)。

在多代理系统中，代理之间需要相互通信。它们通过[交接（handoffs）](#handoffs)进行通信——这是一种描述将控制权移交给哪个代理以及要发送到该代理的负载（payload）的原语。

本指南涵盖以下内容：

* 实现代理间的[交接（handoffs）](#handoffs)
* 使用交接（handoffs）和预构建的[代理](../agents/agents.md)来[构建自定义多代理系统](#build-a-multi-agent-system)

要开始构建多代理系统，请查看 LangGraph 提供的两个最流行的多代理架构的[预构建实现](#prebuilt-implementations)——[监督者（supervisor）](../agents/multi-agent.md#supervisor)和[蜂群（swarm）](../agents/multi-agent.md#swarm)。

## 交接（Handoffs）

要设置多代理系统中的代理之间的通信，您可以使用[**交接（handoffs）**](../concepts/multi_agent.md#handoffs)——一种代理*交接*控制权给另一个代理的模式。交接允许您指定：

- **目标（destination）**：要导航到的目标代理（例如，LangGraph 中要去的节点名称）
- **负载（payload）**：要传递给该代理的信息（例如，状态更新）

### 创建交接（Handoffs）

要实现交接，您可以从代理节点或工具返回`Command`对象：

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
        return Command( # (2)!
            # highlight-next-line
            goto=agent_name, # (3)!
            # highlight-next-line
            update={"messages": state["messages"] + [tool_message]}, # (4)!
            # highlight-next-line
            graph=Command.PARENT, # (5)!
        )
    return handoff_tool
```

1. 使用 [InjectedState][langgraph.prebuilt.InjectedState] 注释访问调用交接工具的代理的[状态（state）](../concepts/low_level.md#state)。
2. `Command` 原语允许将状态更新和节点转换作为单个操作进行指定，这对于实现交接非常有用。
3. 要交接到的代理或节点的名称。
4. 获取代理的消息并将其作为交接的一部分**添加到**父级的**状态（state）**中。下一个代理将看到父级状态。
5. 指示 LangGraph 我们需要导航到**父级**多代理图中的代理节点。

!!! tip

    如果您想使用返回 `Command` 的工具，您可以要么使用预构建的 [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent] / [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode] 组件，要么实现自己的工具执行节点，该节点收集工具返回的 `Command` 对象并返回一个列表，例如：

    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```

!!! Important

    此交接实现假定：

      - 每个代理在多代理系统中都接收整体消息历史（跨所有代理）作为其输入。如果您想更精确地控制代理输入，请参阅[此部分](#control-agent-inputs)
      - 每个代理将其内部消息历史输出到多代理系统的整体消息历史中。如果您想更精确地控制**如何添加代理输出**，请将代理包装在一个单独的节点函数中：

        ```python
        def call_hotel_assistant(state):
            # 返回代理的最终响应，
            # 排除内部思考过程
            response = hotel_assistant.invoke(state)
            # highlight-next-line
            return {"messages": response["messages"][-1]}
        ```

### 控制代理输入

您可以使用 [`Send()`][langgraph.types.Send] 原语在交接过程中将数据直接发送到工作代理。例如，您可以要求调用代理为下一个代理填充任务描述：

```python

from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import InjectedState
from langgraph.graph import StateGraph, START, MessagesState
# highlight-next-line
from langgraph.types import Command, Send

def create_task_description_handoff_tool(
    *, agent_name: str, description: str | None = None
):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        # this is populated by the calling agent
        task_description: Annotated[
            str,
            "Description of what the next agent should do, including all of the relevant context.",
        ],
        # these parameters are ignored by the LLM
        state: Annotated[MessagesState, InjectedState],
    ) -> Command:
        task_description_message = {"role": "user", "content": task_description}
        agent_input = {**state, "messages": [task_description_message]}
        return Command(
            # highlight-next-line
            goto=[Send(agent_name, agent_input)],
            graph=Command.PARENT,
        )

    return handoff_tool
```

请参阅多代理[监督者（supervisor）](../tutorials/multi_agent/agent_supervisor.md#4-create-delegation-tasks)示例，了解使用交接中的 [`Send()`][langgraph.types.Send] 的完整示例。

## 构建多代理系统

您可以在使用 LangGraph 构建的任何代理中使用交接。我们建议使用预构建的[代理](../agents/overview.md)或[`ToolNode`](./tool-calling.md#toolnode)，因为它们原生支持返回 `Command` 的交接工具。以下是一个如何使用交接实现预订旅行的多代理系统的示例：

```python
from langgraph.prebuilt import create_react_agent
from langgraph.graph import StateGraph, START, MessagesState

def create_handoff_tool(*, agent_name: str, description: str | None = None):
    # same implementation as above
    ...
    return Command(...)

# Handoffs
transfer_to_hotel_assistant = create_handoff_tool(agent_name="hotel_assistant")
transfer_to_flight_assistant = create_handoff_tool(agent_name="flight_assistant")

# Define agents
flight_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[..., transfer_to_hotel_assistant],
    # highlight-next-line
    name="flight_assistant"
)
hotel_assistant = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    # highlight-next-line
    tools=[..., transfer_to_flight_assistant],
    # highlight-next-line
    name="hotel_assistant"
)

# Define multi-agent graph
multi_agent_graph = (
    StateGraph(MessagesState)
    # highlight-next-line
    .add_node(flight_assistant)
    # highlight-next-line
    .add_node(hotel_assistant)
    .add_edge(START, "flight_assistant")
    .compile()
)
```

??? example "完整示例：预订旅行的多代理系统"

    ```python
    from typing import Annotated
    from langchain_core.messages import convert_to_messages
    from langchain_core.tools import tool, InjectedToolCallId
    from langgraph.prebuilt import create_react_agent, InjectedState
    from langgraph.graph import StateGraph, START, MessagesState
    from langgraph.types import Command
    
    # We'll use `pretty_print_messages` helper to render the streamed agent outputs nicely
    
    def pretty_print_message(message, indent=False):
        pretty_message = message.pretty_repr(html=True)
        if not indent:
            print(pretty_message)
            return
    
        indented = "\n".join("\t" + c for c in pretty_message.split("\n"))
        print(indented)
    
    
    def pretty_print_messages(update, last_message=False):
        is_subgraph = False
        if isinstance(update, tuple):
            ns, update = update
            # skip parent graph updates in the printouts
            if len(ns) == 0:
                return
    
            graph_id = ns[-1].split(":")[0]
            print(f"Update from subgraph {graph_id}:")
            print("\n")
            is_subgraph = True
    
        for node_name, node_update in update.items():
            update_label = f"Update from node {node_name}:"
            if is_subgraph:
                update_label = "\t" + update_label
    
            print(update_label)
            print("\n")
    
            messages = convert_to_messages(node_update["messages"])
            if last_message:
                messages = messages[-1:]
    
            for m in messages:
                pretty_print_message(m, indent=is_subgraph)
            print("\n")


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
            return Command( # (2)!
                # highlight-next-line
                goto=agent_name, # (3)!
                # highlight-next-line
                update={"messages": state["messages"] + [tool_message]}, # (4)!
                # highlight-next-line
                graph=Command.PARENT, # (5)!
            )
        return handoff_tool
    
    # Handoffs
    transfer_to_hotel_assistant = create_handoff_tool(
        agent_name="hotel_assistant",
        description="Transfer user to the hotel-booking assistant.",
    )
    transfer_to_flight_assistant = create_handoff_tool(
        agent_name="flight_assistant",
        description="Transfer user to the flight-booking assistant.",
    )
    
    # Simple agent tools
    def book_hotel(hotel_name: str):
        """Book a hotel"""
        return f"Successfully booked a stay at {hotel_name}."
    
    def book_flight(from_airport: str, to_airport: str):
        """Book a flight"""
        return f"Successfully booked a flight from {from_airport} to {to_airport}."
    
    # Define agents
    flight_assistant = create_react_agent(
        model="anthropic:claude-3-5-sonnet-latest",
        # highlight-next-line
        tools=[book_flight, transfer_to_hotel_assistant],
        prompt="You are a flight booking assistant",
        # highlight-next-line
        name="flight_assistant"
    )
    hotel_assistant = create_react_agent(
        model="anthropic:claude-3-5-sonnet-latest",
        # highlight-next-line
        tools=[book_hotel, transfer_to_flight_assistant],
        prompt="You are a hotel booking assistant",
        # highlight-next-line
        name="hotel_assistant"
    )
    
    # Define multi-agent graph
    multi_agent_graph = (
        StateGraph(MessagesState)
        .add_node(flight_assistant)
        .add_node(hotel_assistant)
        .add_edge(START, "flight_assistant")
        .compile()
    )
    
    # Run the multi-agent graph
    for chunk in multi_agent_graph.stream(
        {
            "messages": [
                {
                    "role": "user",
                    "content": "book a flight from BOS to JFK and a stay at McKittrick Hotel"
                }
            ]
        },
        # highlight-next-line
        subgraphs=True
    ):
        pretty_print_messages(chunk)
    ```

    1. Access agent's state
    2. The `Command` primitive allows specifying a state update and a node transition as a single operation, making it useful for implementing handoffs.
    3. الاسم للوكيل أو العقدة التي سيتم تسليمها إليها.
    4. 获取代理的消息并将其**添加**到父级的**状态（state）**中作为交接的一部分。下一个代理将看到父级状态。
    5. 指示 LangGraph 我们需要导航到**父级**多代理图中的代理节点。

## 多轮对话

用户可能希望与一个或多个代理进行*多轮对话*。要构建能够处理此问题的系统，您可以创建一个使用[`interrupt`][langgraph.types.interrupt] 来收集用户输入并路由回*活动*代理的节点。

然后，可以将代理实现为图中执行代理步骤并确定下一步操作的节点：

1. **等待用户输入**以继续对话，或者
2. **通过交接（handoff）路由到另一个代理**（或返回到自身，例如在循环中）

```python
def human(state) -> Command[Literal["agent", "another_agent"]]:
    """A node for collecting user input."""
    user_input = interrupt(value="Ready for user input.")

    # Determine the active agent.
    active_agent = ...

    ...
    return Command(
        update={
            "messages": [{
                "role": "human",
                "content": user_input,
            }]
        },
        goto=active_agent
    )

def agent(state) -> Command[Literal["agent", "another_agent", "human"]]:
    # The condition for routing/halting can be anything, e.g. LLM tool call / structured output, etc.
    goto = get_next_agent(...)  # 'agent' / 'another_agent'
    if goto:
        return Command(goto=goto, update={"my_state_key": "my_state_value"})
    else:
        return Command(goto="human") # Go to human node
```

??? example "完整示例：旅行推荐多代理系统"

    在此示例中，我们将构建一个可以经由交接相互通信的旅行助手代理团队。

    我们将创建 2 个代理：

    * travel_advisor：可以帮助进行旅行目的地推荐。可以向 hotel_advisor 求助。
    * hotel_advisor：可以帮助进行酒店推荐。可以向 travel_advisor 求助。

    ```python
    from langchain_anthropic import ChatAnthropic
    from langgraph.graph import MessagesState, StateGraph, START
    from langgraph.prebuilt import create_react_agent, InjectedState
    from langgraph.types import Command, interrupt
    from langgraph.checkpoint.memory import MemorySaver
    
    
    model = ChatAnthropic(model="claude-3-5-sonnet-latest")

    class MultiAgentState(MessagesState):
        last_active_agent: str
    
    
    # Define travel advisor tools and ReAct agent
    travel_advisor_tools = [
        get_travel_recommendations,
        make_handoff_tool(agent_name="hotel_advisor"),
    ]
    travel_advisor = create_react_agent(
        model,
        travel_advisor_tools,
        prompt=(
            "You are a general travel expert that can recommend travel destinations (e.g. countries, cities, etc). "
            "If you need hotel recommendations, ask 'hotel_advisor' for help. "
            "You MUST include human-readable response before transferring to another agent."
        ),
    )
    
    
    def call_travel_advisor(
        state: MultiAgentState,
    ) -> Command[Literal["hotel_advisor", "human"]]:
        # You can also add additional logic like changing the input to the agent / output from the agent, etc.
        # NOTE: we're invoking the ReAct agent with the full history of messages in the state
        response = travel_advisor.invoke(state)
        update = {**response, "last_active_agent": "travel_advisor"}
        return Command(update=update, goto="human")
    
    
    # Define hotel advisor tools and ReAct agent
    hotel_advisor_tools = [
        get_hotel_recommendations,
        make_handoff_tool(agent_name="travel_advisor"),
    ]
    hotel_advisor = create_react_agent(
        model,
        hotel_advisor_tools,
        prompt=(
            "You are a hotel expert that can provide hotel recommendations for a given destination. "
            "If you need help picking travel destinations, ask 'travel_advisor' for help."
            "You MUST include human-readable response before transferring to another agent."
        ),
    )
    
    
    def call_hotel_advisor(
        state: MultiAgentState,
    ) -> Command[Literal["travel_advisor", "human"]]:
        response = hotel_advisor.invoke(state)
        update = {**response, "last_active_agent": "hotel_advisor"}
        return Command(update=update, goto="human")
    
    
    def human_node(
        state: MultiAgentState, config
    ) -> Command[Literal["hotel_advisor", "travel_advisor", "human"]]:
        """A node for collecting user input."""
    
        user_input = interrupt(value="Ready for user input.")
        active_agent = state["last_active_agent"]
    
        return Command(
            update={
                "messages": [
                    {
                        "role": "human",
                        "content": user_input,
                    }
                ]
            },
            goto=active_agent,
        )
    
    
    builder = StateGraph(MultiAgentState)
    builder.add_node("travel_advisor", call_travel_advisor)
    builder.add_node("hotel_advisor", call_hotel_advisor)
    
    # This adds a node to collect human input, which will route
    # back to the active agent.
    builder.add_node("human", human_node)
    
    # We'll always start with a general travel advisor.
    builder.add_edge(START, "travel_advisor")
    
    
    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)
    ```
    
    Let's test a multi turn conversation with this application.

    ```python
    import uuid
    
    thread_config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    
    inputs = [
        # 1st round of conversation,
        {
            "messages": [
                {"role": "user", "content": "i wanna go somewhere warm in the caribbean"}
            ]
        },
        # Since we're using `interrupt`, we'll need to resume using the Command primitive.
        # 2nd round of conversation,
        Command(
            resume="could you recommend a nice hotel in one of the areas and tell me which area it is."
        ),
        # 3rd round of conversation,
        Command(
            resume="i like the first one. could you recommend something to do near the hotel?"
        ),
    ]
    
    for idx, user_input in enumerate(inputs):
        print()
        print(f"--- Conversation Turn {idx + 1} ---")
        print()
        print(f"User: {user_input}")
        print()
        for update in graph.stream(
            user_input,
            config=thread_config,
            stream_mode="updates",
        ):
            for node_id, value in update.items():
                if isinstance(value, dict) and value.get("messages", []):
                    last_message = value["messages"][-1]
                    if isinstance(last_message, dict) or last_message.type != "ai":
                        continue
                    print(f"{node_id}: {last_message.content}")
    ```
    
    ```
    --- Conversation Turn 1 ---
    
    User: {'messages': [{'role': 'user', 'content': 'i wanna go somewhere warm in the caribbean'}]}
    
    travel_advisor: Based on the recommendations, Aruba would be an excellent choice for your Caribbean getaway! Aruba is known as "One Happy Island" and offers:
    - Year-round warm weather with consistent temperatures around 82°F (28°C)
    - Beautiful white sand beaches like Eagle Beach and Palm Beach
    - Clear turquoise waters perfect for swimming and snorkeling
    - Minimal rainfall and location outside the hurricane belt
    - A blend of Caribbean and Dutch culture
    - Great dining options and nightlife
    - Various water sports and activities
    
    Would you like me to get some specific hotel recommendations in Aruba for your stay? I can transfer you to our hotel advisor who can help with accommodations.
    
    --- Conversation Turn 2 ---
    
    User: Command(resume='could you recommend a nice hotel in one of the areas and tell me which area it is.')
    
    hotel_advisor: Based on the recommendations, I can suggest two excellent options:
    
    1. The Ritz-Carlton, Aruba - Located in Palm Beach
    - This luxury resort is situated in the vibrant Palm Beach area
    - Known for its exceptional service and amenities
    - Perfect if you want to be close to dining, shopping, and entertainment
    - Features multiple restaurants, a casino, and a world-class spa
    - Located on a pristine stretch of Palm Beach
    
    2. Bucuti & Tara Beach Resort - Located in Eagle Beach
    - An adults-only boutique resort on Eagle Beach
    - Known for being more intimate and peaceful
    - Award-winning for its sustainability practices
    - Perfect for a romantic getaway or peaceful vacation
    - Located on one of the most beautiful beaches in the Caribbean
    
    Would you like more specific information about either of these properties or their locations?
    
    --- Conversation Turn 3 ---
    
    User: Command(resume='i like the first one. could you recommend something to do near the hotel?')
    
    travel_advisor: Near the Ritz-Carlton in Palm Beach, here are some highly recommended activities:
    
    1. Visit the Palm Beach Plaza Mall - Just a short walk from the hotel, featuring shopping, dining, and entertainment
    2. Try your luck at the Stellaris Casino - It's right in the Ritz-Carlton
    3. Take a sunset sailing cruise - Many depart from the nearby pier
    4. Visit the California Lighthouse - A scenic landmark just north of Palm Beach
    5. Enjoy water sports at Palm Beach:
       - Jet skiing
       - Parasailing
       - Snorkeling
       - Stand-up paddleboarding
    
    Would you like more specific information about any of these activities or would you like to know about other options in the area?
    ```

## 预构建实现

LangGraph 提供了两个最受欢迎的多代理架构的预构建实现：

- [监督者（supervisor）](../agents/multi-agent.md#supervisor) — 单个代理由一个中央监督者代理协调。监督者控制所有通信流和任务委派，根据当前上下文和任务要求决定调用哪个代理。您可以使用 [`langgraph-supervisor`](https://github.com/langchain-ai/langgraph-supervisor-py) 库来创建监督者多代理系统。
- [蜂群（swarm）](../agents/multi-agent.md#supervisor) — 代理根据其专业领域动态地将控制权交接给彼此。系统会记住最后活动的代理，确保在后续交互中，对话能继续与该代理进行。您可以使用 [`langgraph-swarm`](https://github.com/langchain-ai/langgraph-swarm-py) 库来创建蜂群多代理系统。