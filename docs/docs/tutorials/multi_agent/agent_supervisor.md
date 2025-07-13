# 多代理主管

[**主管**](../../concepts/multi_agent.md#supervisor) 是一种多代理架构，其中**专业**代理由一个中央**主管代理**进行协调。主管代理控制所有通信流程和任务委派，根据当前上下文和任务要求决定调用哪个代理。

在本教程中，您将构建一个包含两个代理——研究专家和数学专家——的主管系统。在本教程结束时，您将：

1. 构建专业的研究和数学代理
2. 使用预构建的 [`langgraph-supervisor`](https://langchain-ai.github.io/langgraph/agents/multi-agent/#supervisor) 构建一个用于编排它们的主管
3. 从头开始构建一个主管
4. 实现高级任务委派

![diagram](assets/diagram.png)

## 设置

首先，我们安装所需的包并设置我们的 API 密钥

```python
%%capture --no-stderr
%pip install -U langgraph langgraph-supervisor langchain-tavily "langchain[openai]"
```

```python
import getpass
import os


def _set_if_undefined(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"Please provide your {var}")


_set_if_undefined("OPENAI_API_KEY")
_set_if_undefined("TAVILY_API_KEY")
```

!!! tip
    注册 LangSmith，以便快速发现问题并提高 LangGraph 项目的性能。[LangSmith](https://docs.smith.langchain.com) 允许您使用追踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用。

## 1. 创建工作代理

首先，我们创建专业的 worker 代理——研究代理和数学代理：

* 研究代理将通过 [Tavily API](https://tavily.com/) 访问网络搜索工具
* 数学代理将访问简单的数学工具（`add`、`multiply`、`divide`）

### 研究代理

对于网络搜索，我们将使用 `langchain-tavily` 中的 `TavilySearch` 工具：

```python
from langchain_tavily import TavilySearch

web_search = TavilySearch(max_results=3)
web_search_results = web_search.invoke("who is the mayor of NYC?")

print(web_search_results["results"][0]["content"])
```

**输出：**
```
Find events, attractions, deals, and more at nyctourism.com Skip Main Navigation Menu The Official Website of the City of New York Text Size Powered by Translate SearchSearch Primary Navigation The official website of NYC Home NYC Resources NYC311 Office of the Mayor Events Connect Jobs Search Office of the Mayor | Mayor's Bio | City of New York Secondary Navigation MayorBiographyNewsOfficials Eric L. Adams 110th Mayor of New York City Mayor Eric Adams has served the people of New York City as an NYPD officer, State Senator, Brooklyn Borough President, and now as the 110th Mayor of the City of New York. Mayor Eric Adams has served the people of New York City as an NYPD officer, State Senator, Brooklyn Borough President, and now as the 110th Mayor of the City of New York. He gave voice to a diverse coalition of working families in all five boroughs and is leading the fight to bring back New York City's economy, reduce inequality, improve public safety, and build a stronger, healthier city that delivers for all New Yorkers. As the representative of one of the nation's largest counties, Eric fought tirelessly to grow the local economy, invest in schools, reduce inequality, improve public safety, and advocate for smart policies and better government that delivers for all New Yorkers.
```

要创建单独的工作代理，我们将使用 LangGraph 的预构建代理 [`agent`](../../agents/agents.md)。

```python
from langgraph.prebuilt import create_react_agent

research_agent = create_react_agent(
    model="openai:gpt-4.1",
    tools=[web_search],
    prompt=(
        "You are a research agent.\n\n"
        "INSTRUCTIONS:\n"
        "- Assist ONLY with research-related tasks, DO NOT do any math\n"
        "- After you're done with your tasks, respond to the supervisor directly\n"
        "- Respond ONLY with the results of your work, do NOT include ANY other text."
    ),
    name="research_agent",
)
```

让我们 [运行代理](../../agents/run_agents.md) 来验证其行为是否符合预期。

!!! note "我们将使用 `pretty_print_messages` 辅助函数来漂亮地渲染流式代理输出"

    ```python
    from langchain_core.messages import convert_to_messages
    
    
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
    ```

```python
from langchain_core.messages import convert_to_messages


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
```

```python
for chunk in research_agent.stream(
    {"messages": [{"role": "user", "content": "who is the mayor of NYC?"}]}
):
    pretty_print_messages(chunk)
```

**输出：**
```
Update from node agent:


================================== Ai Message ==================================
Name: research_agent
Tool Calls:
  tavily_search (call_U748rQhQXT36sjhbkYLSXQtJ)
 Call ID: call_U748rQhQXT36sjhbkYLSXQtJ
  Args:
    query: current mayor of New York City


Update from node tools:


================================= Tool Message ==================================
Name: tavily_search

{"query": "current mayor of New York City", "follow_up_questions": null, "answer": null, "images": [], "results": [{"title": "List of mayors of New York City - Wikipedia", "url": "https://en.wikipedia.org/wiki/List_of_mayors_of_New_York_City", "content": "The mayor of New York City is the chief executive of the Government of New York City, as stipulated by New York City's charter.The current officeholder, the 110th in the sequence of regular mayors, is Eric Adams, a member of the Democratic Party.. During the Dutch colonial period from 1624 to 1664, New Amsterdam was governed by the Director of Netherland.", "score": 0.9039154, "raw_content": null}, {"title": "Office of the Mayor | Mayor's Bio | City of New York - NYC.gov", "url": "https://www.nyc.gov/office-of-the-mayor/bio.page", "content": "Mayor Eric Adams has served the people of New York City as an NYPD officer, State Senator, Brooklyn Borough President, and now as the 110th Mayor of the City of New York. He gave voice to a diverse coalition of working families in all five boroughs and is leading the fight to bring back New York City's economy, reduce inequality, improve", "score": 0.8405867, "raw_content": null}, {"title": "Eric Adams - Wikipedia", "url": "https://en.wikipedia.org/wiki/Eric_Adams", "content": "Eric Leroy Adams (born September 1, 1960) is an American politician and former police officer who has served as the 110th mayor of New York City since 2022. Adams was an officer in the New York City Transit Police and then the New York City Police Department (```
```

### 数学代理

对于数学代理工具，我们将使用 [原生 Python 函数](../../how-tos/tool-calling.md#define-a-tool)：

```python
def add(a: float, b: float):
    """Add two numbers."""
    return a + b


def multiply(a: float, b: float):
    """Multiply two numbers."""
    return a * b


def divide(a: float, b: float):
    """Divide two numbers."""
    return a / b


math_agent = create_react_agent(
    model="openai:gpt-4.1",
    tools=[add, multiply, divide],
    prompt=(
        "You are a math agent.\n\n"
        "INSTRUCTIONS:\n"
        "- Assist ONLY with math-related tasks\n"
        "- After you're done with your tasks, respond to the supervisor directly\n"
        "- Respond ONLY with the results of your work, do NOT include ANY other text."
    ),
    name="math_agent",
)
```

让我们运行数学代理：

```python
for chunk in math_agent.stream(
    {"messages": [{"role": "user", "content": "what's (3 + 5) x 7"}]}
):
    pretty_print_messages(chunk)
```

**输出：**
```
Update from node agent:


================================== Ai Message ==================================
Name: math_agent
Tool Calls:
  add (call_p6OVLDHB4LyCNCxPOZzWR15v)
 Call ID: call_p6OVLDHB4LyCNCxPOZzWR15v
  Args:
    a: 3
    b: 5


Update from node tools:


================================= Tool Message ==================================
Name: add

8.0


Update from node agent:


================================== Ai Message ==================================
Name: math_agent
Tool Calls:
  multiply (call_EoaWHMLFZAX4AkajQCtZvbli)
 Call ID: call_EoaWHMLFZAX4AkajQCtZvbli
  Args:
    a: 8
    b: 7


Update from node tools:


================================= Tool Message ==================================
Name: multiply

56.0


Update from node agent:


================================== Ai Message ==================================
Name: math_agent

56


```

## 2. 使用 `langgraph-supervisor` 创建主管

要实现我们的多代理系统，我们将使用 `langgraph-supervisor` 库中的 [`create_supervisor`][langgraph_supervisor.supervisor.create_supervisor]：

```python
from langgraph_supervisor import create_supervisor
from langchain.chat_models import init_chat_model

supervisor = create_supervisor(
    model=init_chat_model("openai:gpt-4.1"),
    agents=[research_agent, math_agent],
    prompt=(
        "You are a supervisor managing two agents:\n"
        "- a research agent. Assign research-related tasks to this agent\n"
        "- a math agent. Assign math-related tasks to this agent\n"
        "Assign work to one agent at a time, do not call agents in parallel.\n"
        "Do not do any work yourself."
    ),
    add_handoff_back_messages=True,
    output_mode="full_history",
).compile()
```

```python
from IPython.display import display, Image

display(Image(supervisor.get_graph().draw_mermaid_png()))
```

![Graph](assets/output.png)

**注意：** 运行此代码时，它将生成并显示主管图的视觉表示，显示主管和工作代理之间的流程。

现在让我们用需要两个代理的查询来运行它：

* 研究代理将查找必要的 GDP 信息
* 数学代理将执行除法以查找纽约州 GDP 的百分比，如下所述

```python
for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "find US and New York state GDP in 2024. what % of US GDP was New York state?",
            }
        ]
    },
):
    pretty_print_messages(chunk, last_message=True)

final_message_history = chunk["supervisor"]["messages"]
```

**输出：**
```
Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_research_agent

Successfully transferred to research_agent


Update from node research_agent:


================================= Tool Message ==================================
Name: transfer_back_to_supervisor

Successfully transferred back to supervisor


Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_math_agent

Successfully transferred to math_agent


Update from node math_agent:


================================= Tool Message ==================================
Name: transfer_back_to_supervisor

Successfully transferred back to supervisor


Update from node supervisor:


================================== Ai Message ==================================
Name: supervisor

In 2024, the US GDP was $29.18 trillion and New York State's GDP was $2.297 trillion. New York State accounted for approximately 7.87% of the total US GDP in 2024.


```

## 3. 从头开始构建主管

现在，让我们从头开始实现相同的多代理系统。我们将需要：

1. [设置主管如何与单个代理通信](#set-up-agent-communication)
2. [创建主管代理](#create-supervisor-agent)
3. 将主管和工作代理合并为[单个多代理图](#create-multi-agent-graph)。

### 设置代理通信

我们需要定义一种主管代理与工作代理通信的方式。在多代理体系结构中实现此目的的一种常见方法是使用**交接 (handoffs)**，其中一个代理将控制权“交接”给另一个代理。交接允许您指定：

- **destination**：要传输到的目标代理
- **payload**：要传递给该代理的信息

我们将通过**交接工具**实现交接，并将这些工具提供给主管代理：当主管调用这些工具时，它将把控制权交接给工作代理，并将完整的消息历史传递给该代理。

```python
from typing import Annotated
from langchain_core.tools import tool, InjectedToolCallId
from langgraph.prebuilt import InjectedState
from langgraph.graph import StateGraph, START, MessagesState
from langgraph.types import Command


def create_handoff_tool(*, agent_name: str, description: str | None = None):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        state: Annotated[MessagesState, InjectedState],
        tool_call_id: Annotated[str, InjectedToolCallId],
    ) -> Command:
        tool_message = {
            "role": "tool",
            "content": f"Successfully transferred to {agent_name}",
            "name": name,
            "tool_call_id": tool_call_id,
        }
        # highlight-next-line
        return Command(
            # highlight-next-line
            goto=agent_name,  # (1)!
            # highlight-next-line
            update={**state, "messages": state["messages"] + [tool_message]},  # (2)!
            # highlight-next-line
            graph=Command.PARENT,  # (3)!
        )

    return handoff_tool


# Handoffs
assign_to_research_agent = create_handoff_tool(
    agent_name="research_agent",
    description="Assign task to a researcher agent.",
)

assign_to_math_agent = create_handoff_tool(
    agent_name="math_agent",
    description="Assign task to a math agent.",
)
```

1. 要交接的代理或节点的名称。
2. 获取代理的消息，并将它们作为交接的一部分添加到父级的状态中。下一个代理将看到父级状态。
3. 指示 LangGraph 我们需要在**父级**多代理图中导航到代理节点。

### 创建主管代理

然后，我们使用刚刚定义的交接工具创建主管代理。我们将使用预构建的 [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent]：

```python
supervisor_agent = create_react_agent(
    model="openai:gpt-4.1",
    tools=[assign_to_research_agent, assign_to_math_agent],
    prompt=(
        "You are a supervisor managing two agents:\n"
        "- a research agent. Assign research-related tasks to this agent\n"
        "- a math agent. Assign math-related tasks to this agent\n"
        "Assign work to one agent at a time, do not call agents in parallel.\n"
        "Do not do any work yourself."
    ),
    name="supervisor",
)
```

### 创建多代理图

将所有这些放在一起，让我们为整个多代理系统创建一个图。我们将主管和单个代理添加为子图[节点](../../concepts/low_level.md#nodes)。

```python
from langgraph.graph import END

# Define the multi-agent supervisor graph
supervisor = (
    StateGraph(MessagesState)
    # NOTE: `destinations` is only needed for visualization and doesn't affect runtime behavior
    .add_node(supervisor_agent, destinations=("research_agent", "math_agent", END))
    .add_node(research_agent)
    .add_node(math_agent)
    .add_edge(START, "supervisor")
    # always return back to the supervisor
    .add_edge("research_agent", "supervisor")
    .add_edge("math_agent", "supervisor")
    .compile()
)
```

请注意，我们已将工作代理与主管之间添加了显式的[边](../../concepts/low_level.md#edges)—这意味着它们保证将控制权交还给主管。如果您希望代理直接回复用户（即，将系统变成一个路由器，您可以删除这些边）。

```python
from IPython.display import display, Image

display(Image(supervisor.get_graph().draw_mermaid_png()))
```

![Graph](assets/multi-output.png)

**注意：** 运行此代码时，它将生成并显示多代理主管图的视觉表示，显示主管和工作代理之间的流程。

创建多代理图后，现在运行它！

```python
for chunk in supervisor.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "find US and New York state GDP in 2024. what % of US GDP was New York state?",
            }
        ]
    },
):
    pretty_print_messages(chunk, last_message=True)

final_message_history = chunk["supervisor"]["messages"]
```

**输出：**
```
Update from node supervisor:


================================= Tool Message ==================================
Name: transfer_to_research_agent

Successfully transferred to research_agent


Update from node research_agent:


================================== Ai Message ==================================
Name: research_agent
Tool Calls:
  tavily_search (call_ZOaTVUA6DKrOjWQldLhtrsO2)
 Call ID: call_ZOaTVUA6DKrOjWQldLhtrsO2
  Args:
    query: US GDP 2024 estimate or actual
    search_depth: advanced
  tavily_search (call_QsRAasxW9K03lTlqjuhNLFbZ)
 Call ID: call_QsRAasxW9K03lTlqjuhNLFbZ
  Args:
    query: New York state GDP 2024 estimate or actual
    search_depth: advanced
================================= Tool Message ==================================
Name: tavily_search

{"query": "US GDP 2024 estimate or actual", "follow_up_questions": null, "answer": null, "images": [], "results": [{"url": "https://www.advisorperspectives.com/dshort/updates/2025/05/29/gdp-gross-domestic-product-q1-2025-second-estimate", "title": "Q1 GDP Second Estimate: Real GDP at -0.2%, Higher Than Expected", "content": "> Real gross domestic product (GDP) decreased at an annual rate of 0.2 percent in the first quarter of 2025 (January, February, and March), according to the second estimate released by the U.S. Bureau of Economic Analysis. In the fourth quarter of 2024, real GDP increased 2.4 percent. The decrease in real GDP in the first quarter primarily reflected an increase in imports, which are a subtraction in the calculation of GDP, and a decrease in government spending. These movements were partly [...] by [Harry Mamaysky](https://www.advisor```
```

!!! important
    您可以看到，主管系统附加了所有单个代理的消息（即它们的内部工具调用循环）到完整的消息历史中。这意味着在每个主管回合中，主管代理都会看到完整的历史记录。如果您想要更多地控制：

    * **如何将输入传递给代理**：您可以使用 LangGraph [`Send()`][langgraph.types.Send] 原语在交接过程中直接将数据发送到工作代理。请参阅下面的[任务委派](#4-create-delegation-tasks)示例
    * **如何添加代理输出**：您可以通过将代理包装到单独的节点函数中来控制添加到整体主管消息历史中的代理内部消息历史的数量：

        ```python
        def call_research_agent(state):
            # return agent's final response,
            # excluding inner monologue
            response = research_agent.invoke(state)
            # highlight-next-line
            return {"messages": response["messages"][-1]}
        ```

## 4. 创建委派任务

到目前为止，单个代理依赖于**解释完整的消息历史**来确定它们的任务。另一种方法是要求主管**明确制定任务**。为此，我们可以向 `handoff_tool` 函数添加 `task_description` 参数。

```python
from langgraph.types import Send


def create_task_description_handoff_tool(
    *, agent_name: str, description: str | None = None
):
    name = f"transfer_to_{agent_name}"
    description = description or f"Ask {agent_name} for help."

    @tool(name, description=description)
    def handoff_tool(
        # this is populated by the supervisor LLM
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


assign_to_research_agent_with_description = create_task_description_handoff_tool(
    agent_name="research_agent",
    description="Assign task to a researcher agent.",
)

assign_to_math_agent_with_description = create_task_description_handoff_tool(
    agent_name="math_agent",
    description="Assign task to a math agent.",
)

supervisor_agent_with_description = create_react_agent(
    model="openai:gpt-4.1",
    tools=[
        assign_to_research_agent_with_description,
        assign_to_math_agent_with_description,
    ],
    prompt=(
        "You are a supervisor managing two agents:\n"
        "- a research agent. Assign research-related tasks to this assistant\n"
        "- a math agent. Assign math-related tasks to this assistant\n"
        "Assign work to one agent at a time, do not call agents in parallel.\n"
        "Do not do any work yourself."
    ),
    name="supervisor",
)

supervisor_with_description = (
    StateGraph(MessagesState)
    .add_node(
        supervisor_agent_with_description, destinations=("research_agent", "math_agent")
    )
    .add_node(research_agent)
    .add_node(math_agent)
    .add_edge(START, "supervisor")
    .add_edge("research_agent", "supervisor")
    .add_edge("math_agent", "supervisor")
    .compile()
)
```

!!! note
    我们正在 [`Send()`][langgraph.types.Send] 原语中使用 `handoff_tool`。这意味着每个工作代理不再接收完整的 `supervisor` 图状态作为输入，而是仅接收 `Send` 载荷的内容。在此示例中，我们将任务描述作为单个“人类”消息发送。

让我们现在使用相同的输入查询运行它：

```python
for chunk in supervisor_with_description.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "find US and New York state GDP in 2024. what % of US GDP was New York state?",
            }
        ]
    },
    subgraphs=True,
):
    pretty_print_messages(chunk, last_message=True)
```

**输出：**
```
Update from subgraph supervisor:


	Update from node agent:


	================================== Ai Message ==================================
	Name: supervisor
	Tool Calls:
	  transfer_to_research_agent (call_tk8q8py8qK6MQz6Kj6mijKua)
	 Call ID: call_tk8q8py8qK6MQz6Kj6mijKua
	  Args:
	    task_description: Find the 2024 GDP (Gross Domestic Product) for both the United States and New York state, using the most up-to-date and reputable sources available. Provide both GDP values and cite the data sources.


Update from subgraph research_agent:


	Update from node agent:


	================================== Ai Message ==================================
	Name: research_agent
	Tool Calls:
	  tavily_search (call_KqvhSvOIhAvXNsT6BOwbPlRB)
	 Call ID: call_KqvhSvOIhAvXNsT6BOwbPlRB
	  Args:
	    query: 2024 United States GDP value from a reputable source
	    search_depth: advanced
	  tavily_search (call_kbbAWBc9KwCWKHmM5v04H88t)
	 Call ID: call_kbbAWBc9KwCWKHmM5v04H88t
	  Args:
	    query: 2024 New York state GDP value from a reputable source
	    search_depth: advanced


Update from subgraph research_agent:


	Update from node tools:


	================================= Tool Message ==================================
	Name: tavily_search
	
	{"query": "2024 United States GDP value from a reputable source", "follow_up_questions": null, "answer": null, "images": [], "results": [{"url": "https://www.focus-economics.com/countries/united-states/", "title": "United States Economy Overview - Focus Economics", "content": "The United States' Macroeconomic Analysis:\n------------------------------------------\n\n**Nominal GDP of USD 29,185 billion in 2024.**\n\n**Nominal GDP of USD 29,179 billion in 2024.**\n\n**GDP per capita of USD 86,635 compared to the global average of USD 10,589.**\n\n**GDP per capita of USD 86,652 compared to the global average of USD 10,589.**\n\n**Average real GDP growth of 2.5% over the last decade.**\n\n**Average real GDP growth of ```
```