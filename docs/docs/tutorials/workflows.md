---
search:
    boost: 2
---
# Workflows and Agents

本指南将回顾用于 Agent 系统的常见模式。在描述这些系统时，对“Workflow”（工作流）和“Agent”（代理）进行区分会很有帮助。理解这种区别的一种方式，在 Anthropic 的博客文章 [《构建有效的 Agent》](https://python.langchain.com/docs/integrations/providers/anthropic/) 中得到了很好的解释：

> 工作流是 LLM 和工具通过预定义的代码路径进行编排的系统。
> 另一方面，Agent 是 LLM 动态指导其自身流程和工具使用的系统，并保持对如何完成任务的控制。

下面是可视化这些差异的一种简单方式：

![Agent Workflow](../concepts/img/agent_workflow.png)

在构建 Agent 和工作流时，LangGraph 提供了许多优势，包括持久化、流式输出以及对调试和部署的支持。

## 设置

您可以使用支持结构化输出和工具调用的 [任何聊天模型](https://python.langchain.com/docs/integrations/chat/)。下面，我们展示了安装包、设置 API 密钥以及为 Anthropic 测试结构化输出/工具调用的过程。

??? "安装依赖"

    ```bash
    pip install langchain_core langchain-anthropic langgraph
    ```

初始化一个 LLM

```python
import os
import getpass

from langchain_anthropic import ChatAnthropic

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")

llm = ChatAnthropic(model="claude-3-5-sonnet-latest")
```

## 构建块：增强型 LLM

LLM 具有支持构建工作流和 Agent 的增强功能。这些包括 [结构化输出](https://python.langchain.com/docs/concepts/structured_outputs/) 和 [工具调用](https://python.langchain.com/docs/concepts/tool_calling/)，如下图所示，摘自 Anthropic 关于《构建有效的 Agent》的博客：

![augmented_llm.png](./workflows/img/augmented_llm.png)


```python
# 结构化输出的模式
from pydantic import BaseModel, Field

class SearchQuery(BaseModel):
    search_query: str = Field(None, description="为网络搜索优化的查询。")
    justification: str = Field(
        None, description="为什么此查询与用户的请求相关。"
    )


# 使用模式增强 LLM
structured_llm = llm.with_structured_output(SearchQuery)

# 调用增强型 LLM
output = structured_llm.invoke("Calcium CT score 与高胆固醇有何关系？")

# 定义一个工具
def multiply(a: int, b: int) -> int:
    return a * b

# 使用工具增强 LLM
llm_with_tools = llm.bind_tools([multiply])

# 调用触发工具调用的 LLM
msg = llm_with_tools.invoke("2 乘以 3 是多少？")

# 获取工具调用
msg.tool_calls
```

## Prompt Chaining（提示链）

在提示链中，每次 LLM 调用都会处理前一次调用的输出。

正如 Anthropic 关于《构建有效的 Agent》的博客中所述：

> 提示链将任务分解为一系列步骤，其中每次 LLM 调用都会处理前一次调用的输出。您可以对任何中间步骤添加程序化检查（参见下图中的“gate”），以确保流程仍按计划进行。

> 何时使用此工作流：此工作流非常适合任务可以轻松而清晰地分解为固定子任务的情况。主要目标是通过使每次 LLM 调用成为一个更简单的任务来权衡延迟以获得更高的准确性。

![prompt_chain.png](./workflows/img/prompt_chain.png)

=== "Graph API"

    ```python
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from IPython.display import Image, display


    # Graph state
    class State(TypedDict):
        topic: str
        joke: str
        improved_joke: str
        final_joke: str


    # Nodes
    def generate_joke(state: State):
        """首次调用 LLM 以生成初始笑话"""

        msg = llm.invoke(f"写一个关于 {state['topic']} 的简短笑话")
        return {"joke": msg.content}


    def check_punchline(state: State):
        """检查笑话是否有包袱的 gate 函数"""

        # 简单检查 - 笑话是否包含 "?" 或 "!"
        if "?" in state["joke"] or "!" in state["joke"]:
            return "Pass"
        return "Fail"


    def improve_joke(state: State):
        """第二次调用 LLM 来改进笑话"""

        msg = llm.invoke(f"通过添加文字游戏来让这个笑话更有趣：{state['joke']}")
        return {"improved_joke": msg.content}


    def polish_joke(state: State):
        """第三次调用 LLM 进行最终润色"""

        msg = llm.invoke(f"为这个笑话添加一个意想不到的转折：{state['improved_joke']}")
        return {"final_joke": msg.content}


    # 构建工作流
    workflow = StateGraph(State)

    # 添加节点
    workflow.add_node("generate_joke", generate_joke)
    workflow.add_node("improve_joke", improve_joke)
    workflow.add_node("polish_joke", polish_joke)

    # 添加边以连接节点
    workflow.add_edge(START, "generate_joke")
    workflow.add_conditional_edges(
        "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
    )
    workflow.add_edge("improve_joke", "polish_joke")
    workflow.add_edge("polish_joke", END)

    # 编译
    chain = workflow.compile()

    # 显示工作流
    display(Image(chain.get_graph().draw_mermaid_png()))

    # 调用
    state = chain.invoke({"topic": "cats"})
    print("Initial joke:")
    print(state["joke"])
    print("\n--- --- ---\n")
    if "improved_joke" in state:
        print("Improved joke:")
        print(state["improved_joke"])
        print("\n--- --- ---\n")

        print("Final joke:")
        print(state["final_joke"])
    else:
        print("笑话未能通过质量检测 - 未检测到包袱！")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/a0281fca-3a71-46de-beee-791468607b75/r

    **资源:**

    **LangChain Academy**

    请在此处查看我们关于提示链的课程 [Courses](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/chain.ipynb)。

=== "Functional API"

    ```python
    from langgraph.func import entrypoint, task


    # Tasks
    @task
    def generate_joke(topic: str):
        """首次调用 LLM 以生成初始笑话"""
        msg = llm.invoke(f"写一个关于 {topic} 的简短笑话")
        return msg.content


    def check_punchline(joke: str):
        """检查笑话是否有包袱的 gate 函数"""
        # 简单检查 - 笑话是否包含 "?" 或 "!"
        if "?" in joke or "!" in joke:
            return "Fail"

        return "Pass"


    @task
    def improve_joke(joke: str):
        """第二次调用 LLM 来改进笑话"""
        msg = llm.invoke(f"通过添加文字游戏来让这个笑话更有趣：{joke}")
        return msg.content


    @task
    def polish_joke(joke: str):
        """第三次调用 LLM 进行最终润色"""
        msg = llm.invoke(f"为这个笑话添加一个意想不到的转折：{joke}")
        return msg.content


    @entrypoint()
    def prompt_chaining_workflow(topic: str):
        original_joke = generate_joke(topic).result()
        if check_punchline(original_joke) == "Pass":
            return original_joke

        improved_joke = improve_joke(original_joke).result()
        return polish_joke(improved_joke).result()

    # 调用
    for step in prompt_chaining_workflow.stream("cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/332fa4fc-b6ca-416e-baa3-161625e69163/r

## 并行处理

通过并行处理，LLM 同时处理任务：

> LLM 有时可以同时处理任务，并通过程序化方式聚合它们的输出。此工作流（并行处理）主要有两种变体：分段：将任务分解为并行的独立子任务。投票：多次运行同一任务以获得不同的输出。

> 何时使用此工作流：当可以针对速度进行并行处理划分的子任务，或者当需要多个观点或尝试来获得更高置信度的结果时，并行处理是有效的。对于具有多个考虑因素的复杂任务，当每个考虑因素由单独的 LLM 调用处理时，LLM 通常表现更好，从而可以专注于每个特定方面。

![parallelization.png](./workflows/img/parallelization.png)

=== "Graph API"

    ```python
    # Graph state
    class State(TypedDict):
        topic: str
        joke: str
        story: str
        poem: str
        combined_output: str


    # Nodes
    def call_llm_1(state: State):
        """首次调用 LLM 以生成初始笑话"""

        msg = llm.invoke(f"写一个关于 {state['topic']} 的笑话")
        return {"joke": msg.content}


    def call_llm_2(state: State):
        """第二次调用 LLM 来生成故事"""

        msg = llm.invoke(f"写一个关于 {state['topic']} 的故事")
        return {"story": msg.content}


    def call_llm_3(state: State):
        """第三次调用 LLM 来生成诗歌"""

        msg = llm.invoke(f"写一首关于 {state['topic']} 的诗")
        return {"poem": msg.content}


    def aggregator(state: State):
        """将笑话和故事合并为一个输出"""

        combined = f"这是关于 {state['topic']} 的故事、笑话和诗！\n\n"
        combined += f"STORY:\n{state['story']}\n\n"
        combined += f"JOKE:\n{state['joke']}\n\n"
        combined += f"POEM:\n{state['poem']}"
        return {"combined_output": combined}


    # 构建工作流
    parallel_builder = StateGraph(State)

    # 添加节点
    parallel_builder.add_node("call_llm_1", call_llm_1)
    parallel_builder.add_node("call_llm_2", call_llm_2)
    parallel_builder.add_node("call_llm_3", call_llm_3)
    parallel_builder.add_node("aggregator", aggregator)

    # 添加边以连接节点
    parallel_builder.add_edge(START, "call_llm_1")
    parallel_builder.add_edge(START, "call_llm_2")
    parallel_builder.add_edge(START, "call_llm_3")
    parallel_builder.add_edge("call_llm_1", "aggregator")
    parallel_builder.add_edge("call_llm_2", "aggregator")
    parallel_builder.add_edge("call_llm_3", "aggregator")
    parallel_builder.add_edge("aggregator", END)
    parallel_workflow = parallel_builder.compile()

    # 显示工作流
    display(Image(parallel_workflow.get_graph().draw_mermaid_png()))

    # 调用
    state = parallel_workflow.invoke({"topic": "cats"})
    print(state["combined_output"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/3be2e53c-ca94-40dd-934f-82ff87fac277/r

    **资源:**

    **Documentation**

    在此处查看我们关于并行处理的文档 [Documentation](https://langchain-ai.github.io/langgraph/how-tos/branching/)。

    **LangChain Academy**

    在此处查看我们关于并行处理的课程 [Courses](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/simple-graph.ipynb)。

=== "Functional API"

    ```python
    @task
    def call_llm_1(topic: str):
        """首次调用 LLM 以生成初始笑话"""
        msg = llm.invoke(f"写一个关于 {topic} 的笑话")
        return msg.content


    @task
    def call_llm_2(topic: str):
        """第二次调用 LLM 来生成故事"""
        msg = llm.invoke(f"写一个关于 {topic} 的故事")
        return msg.content


    @task
    def call_llm_3(topic):
        """第三次调用 LLM 来生成诗歌"""
        msg = llm.invoke(f"写一首关于 {topic} 的诗")
        return msg.content


    @task
    def aggregator(topic, joke, story, poem):
        """将笑话和故事合并为一个输出"""

        combined = f"这是关于 {topic} 的故事、笑话和诗！\n\n"
        combined += f"STORY:\n{story}\n\n"
        combined += f"JOKE:\n{joke}\n\n"
        combined += f"POEM:\n{poem}"
        return combined


    # 构建工作流
    @entrypoint()
    def parallel_workflow(topic: str):
        joke_fut = call_llm_1(topic)
        story_fut = call_llm_2(topic)
        poem_fut = call_llm_3(topic)
        return aggregator(
            topic, joke_fut.result(), story_fut.result(), poem_fut.result()
        ).result()

    # 调用
    for step in parallel_workflow.stream("cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/623d033f-e814-41e9-80b1-75e6abb67801/r

## 路由

路由对输入进行分类并将其定向到后续任务。正如 Anthropic 关于《构建有效的 Agent》的博客中所述：

> 路由对输入进行分类并将其定向到专门的后续任务。此工作流允许关注点分离，并构建更专门的提示。没有此工作流，针对一种输入进行优化会损害对其他输入的性能。

> 何时使用此工作流：路由适用于可以准确处理分类的复杂任务，其中存在需要单独处理的清晰类别。它可以由 LLM 或更传统的分类模型/算法处理。

![routing.png](./workflows/img/routing.png)


=== "Graph API"

    ```python
    from typing_extensions import Literal
    from langchain_core.messages import HumanMessage, SystemMessage


    # 用于路由逻辑的结构化输出模式
    class Route(BaseModel):
        step: Literal["poem", "story", "joke"] = Field(
            None, description="路由过程的下一步"
        )


    # 使用模式增强 LLM
    router = llm.with_structured_output(Route)


    # State
    class State(TypedDict):
        input: str
        decision: str
        output: str


    # Nodes
    def llm_call_1(state: State):
        """写一个故事"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_2(state: State):
        """写一个笑话"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_3(state: State):
        """写一首诗"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_router(state: State):
        """将输入路由到适当的节点"""

        # 使用结构化输出调用增强型 LLM 作为路由逻辑
        decision = router.invoke(
            [
                SystemMessage(
                    content="根据用户请求，将输入路由到故事、笑话或诗歌。"
                ),
                HumanMessage(content=state["input"]),
            ]
        )

        return {"decision": decision.step}


    # 用于将输入路由到适当节点的条件边函数
    def route_decision(state: State):
        # 返回要访问的下一个节点名称
        if state["decision"] == "story":
            return "llm_call_1"
        elif state["decision"] == "joke":
            return "llm_call_2"
        elif state["decision"] == "poem":
            return "llm_call_3"


    # 构建工作流
    router_builder = StateGraph(State)

    # 添加节点
    router_builder.add_node("llm_call_1", llm_call_1)
    router_builder.add_node("llm_call_2", llm_call_2)
    router_builder.add_node("llm_call_3", llm_call_3)
    router_builder.add_node("llm_call_router", llm_call_router)

    # 添加边以连接节点
    router_builder.add_edge(START, "llm_call_router")
    router_builder.add_conditional_edges(
        "llm_call_router",
        route_decision,
        {  # route_decision 返回的名称 : 要访问的下一个节点名称
            "llm_call_1": "llm_call_1",
            "llm_call_2": "llm_call_2",
            "llm_call_3": "llm_call_3",
        },
    )
    router_builder.add_edge("llm_call_1", END)
    router_builder.add_edge("llm_call_2", END)
    router_builder.add_edge("llm_call_3", END)

    # 编译工作流
    router_workflow = router_builder.compile()

    # 显示工作流
    display(Image(router_workflow.get_graph().draw_mermaid_png()))

    # 调用
    state = router_workflow.invoke({"input": "给我写个关于猫的笑话"})
    print(state["output"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/c4580b74-fe91-47e4-96fe-7fac598d509c/r

    **资源:**

    **LangChain Academy**

    在此处查看我们关于路由的课程 [Courses](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/router.ipynb)。

    **Examples**

    [此处](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag_local/) 是一个路由问题的 RAG 工作流。在此处观看我们的视频 [Video](https://www.youtube.com/watch?v=bq1Plo2RhYI)。

=== "Functional API"

    ```python
    from typing_extensions import Literal
    from pydantic import BaseModel
    from langchain_core.messages import HumanMessage, SystemMessage


    # 用于路由逻辑的结构化输出模式
    class Route(BaseModel):
        step: Literal["poem", "story", "joke"] = Field(
            None, description="路由过程的下一步"
        )


    # 使用模式增强 LLM
    router = llm.with_structured_output(Route)


    @task
    def llm_call_1(input_: str):
        """写一个故事"""
        result = llm.invoke(input_)
        return result.content


    @task
    def llm_call_2(input_: str):
        """写一个笑话"""
        result = llm.invoke(input_)
        return result.content


    @task
    def llm_call_3(input_: str):
        """写一首诗"""
        result = llm.invoke(input_)
        return result.content


    def llm_call_router(input_: str):
        """将输入路由到适当的节点"""
        # 使用结构化输出调用增强型 LLM 作为路由逻辑
        decision = router.invoke(
            [
                SystemMessage(
                    content="根据用户请求，将输入路由到故事、笑话或诗歌。"
                ),
                HumanMessage(content=input_),
            ]
        )
        return decision.step


    # 创建工作流
    @entrypoint()
    def router_workflow(input_: str):
        next_step = llm_call_router(input_)
        if next_step == "story":
            llm_call = llm_call_1
        elif next_step == "joke":
            llm_call = llm_call_2
        elif next_step == "poem":
            llm_call = llm_call_3

        return llm_call(input_).result()

    # 调用
    for step in router_workflow.stream("给我写个关于猫的笑话", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/5e2eb979-82dd-402c-b1a0-a8cceaf2a28a/r

## Orchestrator-Worker（协调器-工作器）

在协调器-工作器工作流中，协调器将任务分解并将其委托给工作器。正如 Anthropic 关于《构建有效的 Agent》的博客中所述：

> 在协调器-工作器工作流中，中央 LLM 动态地分解任务，将其委托给工作器 LLM，并综合它们的输出。

> 何时使用此工作流：此工作流非常适合您无法预测所需子任务的复杂任务（例如，在编码中，需要更改的文件数量以及每种文件的更改性质可能取决于任务）。虽然在拓扑上相似，但与并行处理的关键区别在于其灵活性——子任务不是预定义的，而是由协调器根据特定输入确定的。

![worker.png](./workflows/img/worker.png)


=== "Graph API"

    ```python
    from typing import Annotated, List
    import operator


    # 用于计划的结构化输出模式
    class Section(BaseModel):
        name: str = Field(
            description="此报告部分的名称。",
        )
        description: str = Field(
            description="将在本节中介绍的主要主题和概念的简要概述。",
        )


    class Sections(BaseModel):
        sections: List[Section] = Field(
            description="报告的各部分。",
        )


    # 使用模式增强 LLM
    planner = llm.with_structured_output(Sections)
    ```

    **在 LangGraph 中创建工作器**

    由于协调器-工作器工作流很常见，LangGraph **提供了 `Send` API 来支持此功能**。它允许您动态创建工作器节点并将每个节点发送特定的输入。每个工作器都有自己的状态，所有工作器输出都写入一个*共享状态键*，该键可供协调器图访问。这使得协调器可以访问所有工作器输出，并允许它将它们综合为最终输出。如下所示，我们遍历了一个部分列表并将每个部分 `Send` 给一个工作器节点。在此处查看更多文档 [Documentation](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 和此处 [Documentation](https://langchain-ai.github.io/langgraph/concepts/low_level/#send)。

    ```python
    from langgraph.types import Send


    # Graph state
    class State(TypedDict):
        topic: str  # 报告主题
        sections: list[Section]  # 报告部分列表
        completed_sections: Annotated[
            list, operator.add
        ]  # 所有工作器都并行写入此键
        final_report: str  # 最终报告


    # Worker state
    class WorkerState(TypedDict):
        section: Section
        completed_sections: Annotated[list, operator.add]


    # Nodes
    def orchestrator(state: State):
        """生成报告计划的协调器"""

        # 生成查询
        report_sections = planner.invoke(
            [
                SystemMessage(content="生成报告计划。"),
                HumanMessage(content=f"报告主题是：{state['topic']}"),
            ]
        )

        return {"sections": report_sections.sections}


    def llm_call(state: WorkerState):
        """工作器编写报告的某一部分"""

        # 生成部分
        section = llm.invoke(
            [
                SystemMessage(
                    content="编写报告的某一部分，遵循提供的名称和描述。每个部分不要包含前言。使用 markdown 格式。"
                ),
                HumanMessage(
                    content=f"此部分名称为：{state['section'].name} 描述为：{state['section'].description}"
                ),
            ]
        )

        # 将更新后的部分写入已完成的部分
        return {"completed_sections": [section.content]}


    def synthesizer(state: State):
        """从各个部分综合完整报告"""

        # 已完成的部分列表
        completed_sections = state["completed_sections"]

        # 将已完成的部分格式化为字符串，用作最终部分的上下文
        completed_report_sections = "\n\n---\n\n".join(completed_sections)

        return {"final_report": completed_report_sections}


    # 用于创建每个部分编写的 llm_call 工作器的条件边函数
    def assign_workers(state: State):
        """为计划中的每个部分分配一个工作器"""

        # 通过 Send() API 并行启动部分编写
        return [Send("llm_call", {"section": s}) for s in state["sections"]]


    # 构建工作流
    orchestrator_worker_builder = StateGraph(State)

    # 添加节点
    orchestrator_worker_builder.add_node("orchestrator", orchestrator)
    orchestrator_worker_builder.add_node("llm_call", llm_call)
    orchestrator_worker_builder.add_node("synthesizer", synthesizer)

    # 添加边以连接节点
    orchestrator_worker_builder.add_edge(START, "orchestrator")
    orchestrator_worker_builder.add_conditional_edges(
        "orchestrator", assign_workers, ["llm_call"]
    )
    orchestrator_worker_builder.add_edge("llm_call", "synthesizer")
    orchestrator_worker_builder.add_edge("synthesizer", END)

    # 编译工作流
    orchestrator_worker = orchestrator_worker_builder.compile()

    # 显示工作流
    display(Image(orchestrator_worker.get_graph().draw_mermaid_png()))

    # 调用
    state = orchestrator_worker.invoke({"topic": "创建一份关于 LLM 标度定律的报告"})

    from IPython.display import Markdown
    Markdown(state["final_report"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/78cbcfc3-38bf-471d-b62a-b299b144237d/r

    **资源:**

    **LangChain Academy**

    在此处查看我们关于协调器-工作器的课程 [Courses](https://github.com/langchain-ai/langchain-academy/blob/main/module-4/map-reduce.ipynb)。

    **Examples**

    [此处](https://github.com/langchain-ai/report-mAIstro) 是一个项目，它使用协调器-工作器进行报告规划和编写。在此处观看我们的视频 [Video](https://www.youtube.com/watch?v=wSxZ7yFbbas)。


=== "Functional API"

    ```python
    from typing import List


    # 用于计划的结构化输出模式
    class Section(BaseModel):
        name: str = Field(
            description="此报告部分的名称。",
        )
        description: str = Field(
            description="将在本节中介绍的主要主题和概念的简要概述。",
        )


    class Sections(BaseModel):
        sections: List[Section] = Field(
            description="报告的各部分。",
        )


    # 使用模式增强 LLM
    planner = llm.with_structured_output(Sections)


    @task
    def orchestrator(topic: str):
        """生成报告计划的协调器"""
        # 生成查询
        report_sections = planner.invoke(
            [
                SystemMessage(content="生成报告计划。"),
                HumanMessage(content=f"报告主题是：{topic}"),
            ]
        )

        return report_sections.sections


    @task
    def llm_call(section: Section):
        """工作器编写报告的某一部分"""

        # 生成部分
        result = llm.invoke(
            [
                SystemMessage(content="编写报告的某一部分。"),
                HumanMessage(
                    content=f"此部分名称为：{section.name} 描述为：{section.description}"
                ),
            ]
        )

        # 将更新后的部分写入已完成的部分
        return result.content


    @task
    def synthesizer(completed_sections: list[str]):
        """从各个部分综合完整报告"""
        final_report = "\n\n---\n\n".join(completed_sections)
        return final_report


    @entrypoint()
    def orchestrator_worker(topic: str):
        sections = orchestrator(topic).result()
        section_futures = [llm_call(section) for section in sections]
        final_report = synthesizer(
            [section_fut.result() for section_fut in section_futures]
        ).result()
        return final_report

    # 调用
    report = orchestrator_worker.invoke("创建一份关于 LLM 标度定律的报告")
    from IPython.display import Markdown
    Markdown(report)
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/75a636d0-6179-4a12-9836-e0aa571e87c5/r

## Evaluator-Optimizer（评估器-优化器）

在评估器-优化器工作流中，一个 LLM 调用生成响应，而另一个 LLM 在循环中提供评估和反馈：

> 在评估器-优化器工作流中，一个 LLM 调用生成响应，而另一个 LLM 在循环中提供评估和反馈。

> 何时使用此工作流：当我们在评估标准方面有明确的指导方针，并且迭代改进可以提供可衡量的价值时，此工作流尤其有效。良好的匹配度有两个迹象：首先，当人类明确表达反馈时，LLM 响应可以得到明显改进；其次，LLM 可以提供此类反馈。这类似于人类作家在制作精美文档时可能会经历的迭代写作过程。

![evaluator_optimizer.png](./workflows/img/evaluator_optimizer.png)

=== "Graph API"

    ```python
    # Graph state
    class State(TypedDict):
        joke: str
        topic: str
        feedback: str
        funny_or_not: str


    # 用于评估的结构化输出模式
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="决定笑话是否好笑。",
        )
        feedback: str = Field(
            description="如果笑话不好笑，请提供改进的反馈。",
        )


    # 使用模式增强 LLM
    evaluator = llm.with_structured_output(Feedback)


    # Nodes
    def llm_call_generator(state: State):
        """LLM 生成笑话"""

        if state.get("feedback"):
            msg = llm.invoke(
                f"写一个关于 {state['topic']} 的笑话，但请考虑反馈：{state['feedback']}"
            )
        else:
            msg = llm.invoke(f"写一个关于 {state['topic']} 的笑话")
        return {"joke": msg.content}


    def llm_call_evaluator(state: State):
        """LLM 评估笑话"""

        grade = evaluator.invoke(f"给笑话打分 {state['joke']}")
        return {"funny_or_not": grade.grade, "feedback": grade.feedback}


    # 用于根据评估器的反馈将决策路由回笑话生成器或结束的条件边函数
    def route_joke(state: State):
        """根据评估器的反馈决定是路由回笑话生成器还是结束"""

        if state["funny_or_not"] == "funny":
            return "Accepted"
        elif state["funny_or_not"] == "not funny":
            return "Rejected + Feedback"


    # 构建工作流
    optimizer_builder = StateGraph(State)

    # 添加节点
    optimizer_builder.add_node("llm_call_generator", llm_call_generator)
    optimizer_builder.add_node("llm_call_evaluator", llm_call_evaluator)

    # 添加边以连接节点
    optimizer_builder.add_edge(START, "llm_call_generator")
    optimizer_builder.add_edge("llm_call_generator", "llm_call_evaluator")
    optimizer_builder.add_conditional_edges(
        "llm_call_evaluator",
        route_joke,
        {  # route_joke 返回的名称 : 要访问的下一个节点名称
            "Accepted": END,
            "Rejected + Feedback": "llm_call_generator",
        },
    )

    # 编译工作流
    optimizer_workflow = optimizer_builder.compile()

    # 显示工作流
    display(Image(optimizer_workflow.get_graph().draw_mermaid_png()))

    # 调用
    state = optimizer_workflow.invoke({"topic": "Cats"})
    print(state["joke"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/86ab3e60-2000-4bff-b988-9b89a3269789/r

    **资源:**

    **Examples**

    [此处](https://github.com/langchain-ai/local-deep-researcher) 是一个使用评估器-优化器来改进报告的助手。在此处观看我们的视频 [Video](https://www.youtube.com/watch?v=XGuTzHoqlj8)。

    [此处](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag_local/) 是一个对答案进行评分以检测幻觉或错误的 RAG 工作流。在此处观看我们的视频 [Video](https://www.youtube.com/watch?v=bq1Plo2RhYI)。

=== "Functional API"

    ```python
    # 用于评估的结构化输出模式
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="决定笑话是否好笑。",
        )
        feedback: str = Field(
            description="如果笑话不好笑，请提供改进的反馈。",
        )


    # 使用模式增强 LLM
    evaluator = llm.with_structured_output(Feedback)


    # Nodes
    @task
    def llm_call_generator(topic: str, feedback: Feedback):
        """LLM 生成笑话"""
        if feedback:
            msg = llm.invoke(
                f"写一个关于 {topic} 的笑话，但请考虑反馈：{feedback}"
            )
        else:
            msg = llm.invoke(f"写一个关于 {topic} 的笑话")
        return msg.content


    @task
    def llm_call_evaluator(joke: str):
        """LLM 评估笑话"""
        feedback = evaluator.invoke(f"给笑话打分 {joke}")
        return feedback


    @entrypoint()
    def optimizer_workflow(topic: str):
        feedback = None
        while True:
            joke = llm_call_generator(topic, feedback).result()
            feedback = llm_call_evaluator(joke).result()
            if feedback.grade == "funny":
                break

        return joke

    # 调用
    for step in optimizer_workflow.stream("Cats", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/f66830be-4339-4a6b-8a93-389ce5ae27b4/r

## Agent（代理）

Agent 通常实现为 LLM 在循环中根据环境反馈执行操作（通过工具调用）。正如 Anthropic 关于《构建有效的 Agent》的博客中所述：

> Agent 可以处理复杂的任务，但它们的实现通常很简单。它们通常只是 LLM 在循环中根据环境反馈使用工具。因此，清晰周到地设计工具集及其文档至关重要。

> 何时使用 Agent：Agent 可用于开放式问题，这些问题很难或不可能预测所需的步骤数，也无法硬编码固定的路径。LLM 可能会进行多个回合的操作，您必须对其决策有一定的信任。Agent 的自主性使其非常适合在受信任的环境中扩展任务。

![agent.png](./workflows/img/agent.png)


```python
from langchain_core.tools import tool


# 定义工具
@tool
def multiply(a: int, b: int) -> int:
    """将 a 和 b 相乘。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """将 a 和 b 相加。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """将 a 和 b 相除。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a / b


# 使用工具增强 LLM
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)
```

=== "Graph API"

    ```python
    from langgraph.graph import MessagesState
    from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage


    # Nodes
    def llm_call(state: MessagesState):
        """LLM 决定是否调用工具"""

        return {
            "messages": [
                llm_with_tools.invoke(
                    [
                        SystemMessage(
                            content="您是一个乐于助人的助手，负责对一组输入执行算术运算。"
                        )
                    ]
                    + state["messages"]
                )
            ]
        }


    def tool_node(state: dict):
        """执行工具调用"""

        result = []
        for tool_call in state["messages"][-1].tool_calls:
            tool = tools_by_name[tool_call["name"]]
            observation = tool.invoke(tool_call["args"])
            result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
        return {"messages": result}


    # 用于根据 LLM 是否进行了工具调用来路由到工具节点或结束的条件边函数
    def should_continue(state: MessagesState) -> Literal["environment", END]:
        """根据 LLM 是否进行了工具调用来决定是继续循环还是停止"""

        messages = state["messages"]
        last_message = messages[-1]
        # 如果 LLM 进行工具调用，则执行操作
        if last_message.tool_calls:
            return "Action"
        # 否则，我们停止（回复用户）
        return END


    # 构建工作流
    agent_builder = StateGraph(MessagesState)

    # 添加节点
    agent_builder.add_node("llm_call", llm_call)
    agent_builder.add_node("environment", tool_node)

    # 添加边以连接节点
    agent_builder.add_edge(START, "llm_call")
    agent_builder.add_conditional_edges(
        "llm_call",
        should_continue,
        {
            # should_continue 返回的名称 : 要访问的下一个节点名称
            "Action": "environment",
            END: END,
        },
    )
    agent_builder.add_edge("environment", "llm_call")

    # 编译 Agent
    agent = agent_builder.compile()

    # 显示 Agent
    display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

    # 调用
    messages = [HumanMessage(content="将 3 和 4 相加。")]
    messages = agent.invoke({"messages": messages})
    for m in messages["messages"]:
        m.pretty_print()
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/051f0391-6761-4f8c-a53b-22231b016690/r

    **资源:**

    **LangChain Academy**

    在此处查看我们关于 Agent 的课程 [Courses](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/agent.ipynb)。

    **Examples**

    [此处](https://github.com/langchain-ai/memory-agent) 是一个使用工具调用 Agent 来创建/存储长期记忆的项目。

=== "Functional API"

    ```python
    from langgraph.graph import add_messages
    from langchain_core.messages import (
        SystemMessage,
        HumanMessage,
        BaseMessage,
        ToolCall,
    )


    @task
    def call_llm(messages: list[BaseMessage]):
        """LLM 决定是否调用工具"""
        return llm_with_tools.invoke(
            [
                SystemMessage(
                    content="您是一个乐于助人的助手，负责对一组输入执行算术运算。"
                )
            ]
            + messages
        )


    @task
    def call_tool(tool_call: ToolCall):
        """执行工具调用"""
        tool = tools_by_name[tool_call["name"]]
        return tool.invoke(tool_call)


    @entrypoint()
    def agent(messages: list[BaseMessage]):
        llm_response = call_llm(messages).result()

        while True:
            if not llm_response.tool_calls:
                break

            # 执行工具
            tool_result_futures = [
                call_tool(tool_call) for tool_call in llm_response.tool_calls
            ]
            tool_results = [fut.result() for fut in tool_result_futures]
            messages = add_messages(messages, [llm_response, *tool_results])
            llm_response = call_llm(messages).result()

        messages = add_messages(messages, llm_response)
        return messages

    # 调用
    messages = [HumanMessage(content="将 3 和 4 相加。")]
    for chunk in agent.stream(messages, stream_mode="updates"):
        print(chunk)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/42ae8bf9-3935-4504-a081-8ddbcbfc8b2e/r

#### Pre-built（预构建）

LangGraph 还提供了一个**预构建方法**来创建如上定义的 Agent（使用 [`create_react_agent`](langgraph.prebuilt.chat_agent_executor.create_react_agent) 函数）：

https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/

```python
from langgraph.prebuilt import create_react_agent

# 传入：
# (1) 使用工具增强的 LLM
# (2) 工具列表（用于创建工具节点）
pre_built_agent = create_react_agent(llm, tools=tools)

# 显示 Agent
display(Image(pre_built_agent.get_graph().draw_mermaid_png()))

# 调用
messages = [HumanMessage(content="将 3 和 4 相加。")]
messages = pre_built_agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

**LangSmith Trace**

https://smith.langchain.com/public/abab6a44-29f6-4b97-8164-af77413e494d/r

## LangGraph 提供的内容

通过在 LangGraph 中构建上述所有内容，我们获得了以下优势：

### 持久化：人工干预

LangGraph 持久化层支持中断和操作批准（例如，人工干预）。请参阅 LangChain Academy 的[模块 3](https://github.com/langchain-ai/langchain-academy/tree/main/module-3)。

### 持久化：记忆

LangGraph 持久化层支持会话式（短期）记忆和长期记忆。请参阅 LangChain Academy 的[模块 2](https://github.com/langchain-ai/langchain-academy/tree/main/module-2) 和 [模块 5](https://github.com/langchain-ai/langchain-academy/tree/main/module-5)：

### 流式输出

LangGraph 提供多种方式来流式输出工作流/Agent 的输出或中间状态。请参阅 LangChain Academy 的[模块 3](https://github.com/langchain-ai/langchain-academy/blob/main/module-3/streaming-interruption.ipynb)。


### 部署

LangGraph 为部署、可观察性和评估提供了一个简单的入口。请参阅 LangChain Academy 的[模块 6](https://github.com/langchain-ai/langchain-academy/tree/main/module-6)。