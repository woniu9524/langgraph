# 使用子图

本指南将介绍使用 [子图](../concepts/subgraphs.md) 的机制。子图的一个常见应用是构建 [多智能体](../concepts/multi_agent.md) 系统。

添加子图时，需要定义父图和子图如何通信：

* [共享状态模式](#shared-state-schemas) — 父图和子图在其状态 [模式](../concepts/low_level.md#state) 中拥有 **共享状态键**
* [不同的状态模式](#different-state-schemas) — 父图和子图的 [模式](../concepts/low_level.md#state) 中**没有共享状态键**

## 设置

```bash
pip install -U langgraph
```

!!! tip "为 LangGraph 开发设置 LangSmith"
    注册 [LangSmith](https://smith.langchain.com) 以快速发现问题并改进 LangGraph 项目的性能。LangSmith 可让您使用跟踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用 — 在此处详细了解如何开始阅读的更多信息。

## 共享状态模式

常见的情况是父图和子图通过状态 [模式](../concepts/low_level.md#state) 中的共享状态键（通道）进行通信。例如，在 [多智能体](../concepts/multi_agent.md) 系统中，智能体通常通过共享的 [消息](https://langchain-ai.github.io/langgraph/concepts/low_level.md#why-use-messages) 键进行通信。

如果您的子图与父图共享状态键，您可以按照以下步骤将其添加到图中：

1. 定义子图工作流（下方示例中的 `subgraph_builder`）并编译它
2. 在定义父图工作流时，将编译后的子图传递给 `.add_node` 方法

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class State(TypedDict):
    foo: str

# 子图

def subgraph_node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 父图

builder = StateGraph(State)
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

??? example "完整示例：共享状态模式"

    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # 定义子图
    class SubgraphState(TypedDict):
        foo: str  # (1)! 
        bar: str  # (2)!
    
    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}
    
    def subgraph_node_2(state: SubgraphState):
        # 注意此节点正在使用仅在子图中可用的状态键 ('bar')
        # 并正在向共享状态键 ('foo') 发送更新
        return {"foo": state["foo"] + state["bar"]}
    
    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()
    
    # 定义父图
    class ParentState(TypedDict):
        foo: str
    
    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}
    
    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", subgraph)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()
    
    for chunk in graph.stream({"foo": "foo"}):
        print(chunk)
    ```

    1. 此键与父图状态共享
    2. 此键是 `SubgraphState` 的私有键，父图看不到
    
    ```
    {'node_1': {'foo': 'hi! foo'}}
    {'node_2': {'foo': 'hi! foobar'}}
    ```

## 不同的状态模式

对于更复杂的系统，您可能希望定义具有与父图**完全不同模式**（无共享键）的子图。例如，您可能希望为 [多智能体](../concepts/multi_agent.md) 系统中的每个智能体维护一个私有消息历史记录。

如果您的应用程序是这种情况，您需要定义一个**调用子图的函数**。该函数需要在调用子图之前转换输入（父）状态到子图状态，并在返回状态更新之前将结果转换回父状态。

```python
from typing_extensions import TypedDict
from langgraph.graph.state import StateGraph, START

class SubgraphState(TypedDict):
    bar: str

# 子图

def subgraph_node_1(state: SubgraphState):
    return {"bar": "hi! " + state["bar"]}

subgraph_builder = StateGraph(SubgraphState)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 父图

class State(TypedDict):
    foo: str

def call_subgraph(state: State):
    subgraph_output = subgraph.invoke({"bar": state["foo"]})  # (1)!
    return {"foo": subgraph_output["bar"]}  # (2)!

builder = StateGraph(State)
builder.add_node("node_1", call_subgraph)
builder.add_edge(START, "node_1")
graph = builder.compile()
```

1. 将状态转换为子图状态
2. 将响应转换回父状态

??? example "完整示例：不同的状态模式"

    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # 定义子图
    class SubgraphState(TypedDict):
        # 注意这些键均未与父图状态共享
        bar: str
        baz: str
    
    def subgraph_node_1(state: SubgraphState):
        return {"baz": "baz"}
    
    def subgraph_node_2(state: SubgraphState):
        return {"bar": state["bar"] + state["baz"]}
    
    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()
    
    # 定义父图
    class ParentState(TypedDict):
        foo: str
    
    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}
    
    def node_2(state: ParentState):
        response = subgraph.invoke({"bar": state["foo"]})  # (1)!
        return {"foo": response["bar"]}  # (2)!
    
    
    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", node_2)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()
    
    for chunk in graph.stream({"foo": "foo"}, subgraphs=True):
        print(chunk)
    ```

    1. 将状态转换为子图状态
    2. 将响应转换回父状态

    ```
    ((), {'node_1': {'foo': 'hi! foo'}})
    (('node_2:9c36dd0f-151a-cb42-cbad-fa2f851f9ab7',), {'grandchild_1': {'my_grandchild_key': 'hi Bob, how are you'}})
    (('node_2:9c36dd0f-151a-cb42-cbad-fa2f851f9ab7',), {'grandchild_2': {'bar': 'hi! foobaz'}})
    ((), {'node_2': {'foo': 'hi! foobaz'}})
    ```

??? example "完整示例：不同状态模式（两层子图）"

    这是关于两层子图的示例：父图 -> 子图 -> 孙子图。

    ```python
    # 孙子图
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START, END
    
    class GrandChildState(TypedDict):
        my_grandchild_key: str
    
    def grandchild_1(state: GrandChildState) -> GrandChildState:
        # 注意：在此处无法访问子图或父图的键
        return {"my_grandchild_key": state["my_grandchild_key"] + ", how are you"}
    
    
    grandchild = StateGraph(GrandChildState)
    grandchild.add_node("grandchild_1", grandchild_1)
    
    grandchild.add_edge(START, "grandchild_1")
    grandchild.add_edge("grandchild_1", END)
    
    grandchild_graph = grandchild.compile()
    
    # 子图
    class ChildState(TypedDict):
        my_child_key: str
    
    def call_grandchild_graph(state: ChildState) -> ChildState:
        # 注意：在此处无法访问父图或孙子图的键
        grandchild_graph_input = {"my_grandchild_key": state["my_child_key"]}  # (1)!
        grandchild_graph_output = grandchild_graph.invoke(grandchild_graph_input)
        return {"my_child_key": grandchild_graph_output["my_grandchild_key"] + " today?"}  # (2)!
    
    child = StateGraph(ChildState)
    child.add_node("child_1", call_grandchild_graph)  # (3)!
    child.add_edge(START, "child_1")
    child.add_edge("child_1", END)
    child_graph = child.compile()
    
    # 父图
    class ParentState(TypedDict):
        my_key: str
    
    def parent_1(state: ParentState) -> ParentState:
        # 注意：在此处无法访问子图或孙子图的键
        return {"my_key": "hi " + state["my_key"]}
    
    def parent_2(state: ParentState) -> ParentState:
        return {"my_key": state["my_key"] + " bye!"}
    
    def call_child_graph(state: ParentState) -> ParentState:
        child_graph_input = {"my_child_key": state["my_key"]}  # (4)!
        child_graph_output = child_graph.invoke(child_graph_input)
        return {"my_key": child_graph_output["my_child_key"]}  # (5)!
    
    parent = StateGraph(ParentState)
    parent.add_node("parent_1", parent_1)
    parent.add_node("child", call_child_graph)  # (6)!
    parent.add_node("parent_2", parent_2)
    
    parent.add_edge(START, "parent_1")
    parent.add_edge("parent_1", "child")
    parent.add_edge("child", "parent_2")
    parent.add_edge("parent_2", END)
    
    parent_graph = parent.compile()
    
    for chunk in parent_graph.stream({"my_key": "Bob"}, subgraphs=True):
        print(chunk)
    ```

    1. 我们正在将状态从子图状态通道 (`my_child_key`) 转换为子图状态通道 (`my_grandchild_key`)
    2. 我们正在将状态从孙子图状态通道 (`my_grandchild_key`) 转换回子图状态通道 (`my_child_key`)
    3. 这里传递的是函数，而不是仅仅是编译后的图（`grandchild_graph`）
    4. 我们正在将状态从父图状态通道 (`my_key`) 转换为子图状态通道 (`my_child_key`)
    5. 我们正在将状态从子图状态通道 (`my_child_key`) 转换回父图状态通道 (`my_key`)
    6. 这里传递的是函数，而不是仅仅是编译后的图（`child_graph`）

    ```
    ((), {'parent_1': {'my_key': 'hi Bob'}})
    (('child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b', 'child_1:781bb3b1-3971-84ce-810b-acf819a03f9c'), {'grandchild_1': {'my_grandchild_key': 'hi Bob, how are you'}})
    (('child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b',), {'child_1': {'my_child_key': 'hi Bob, how are you today?'}})
    ((), {'child': {'my_key': 'hi Bob, how are you today?'}})
    ((), {'parent_2': {'my_key': 'hi Bob, how are you today? bye!'}})
    ```

## 添加持久化

您只需要在**编译父图时提供 checkpointer**。LangGraph 会自动将 checkpointer 传播到子图。

```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from typing_extensions import TypedDict

class State(TypedDict):
    foo: str

# 子图

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
subgraph = subgraph_builder.compile()

# 父图

builder = StateGraph(State)
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```    

如果您希望子图**拥有自己的内存**，可以通过 `with checkpointer=True` 来编译它。这在 [多智能体](../concepts/multi_agent.md) 系统中很有用，如果您希望智能体能够跟踪它们内部的消息历史记录：

```python
subgraph_builder = StateGraph(...)
subgraph = subgraph_builder.compile(checkpointer=True)
```

## 查看子图状态

当您启用 [持久化](../concepts/persistence.md) 时，可以通过 `graph.get_state(config)` 来 [检查图状态](../concepts/persistence.md#checkpoints)（检查点）。要查看子图状态，可以使用 `graph.get_state(config, subgraphs=True)`。

!!! important "仅在中断时可用"

    子图状态**仅在子图中断时**才能查看。一旦您恢复图，将无法访问子图状态。

??? example "查看中断的子图状态"

    ```python
    from langgraph.graph import START, StateGraph
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.types import interrupt, Command
    from typing_extensions import TypedDict
    
    class State(TypedDict):
        foo: str
    
    # 子图
    
    def subgraph_node_1(state: State):
        value = interrupt("Provide value:")
        return {"foo": state["foo"] + value}
    
    subgraph_builder = StateGraph(State)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    
    subgraph = subgraph_builder.compile()
    
    # 父图
        
    builder = StateGraph(State)
    builder.add_node("node_1", subgraph)
    builder.add_edge(START, "node_1")
    
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)
    
    config = {"configurable": {"thread_id": "1"}}
    
    graph.invoke({"foo": ""}, config)
    parent_state = graph.get_state(config)
    subgraph_state = graph.get_state(config, subgraphs=True).tasks[0].state  # (1)!
    
    # 恢复子图
    graph.invoke(Command(resume="bar"), config)
    ```
    
    1. 只有当子图中断时才能获得此信息。一旦您恢复图，将无法访问子图状态。

## 流式输出子图输出

要将子图的输出包含在流式输出中，可以在父图的 `.stream()` 方法中设置 `subgraphs=True`。这将流式传输父图和任何子图的输出。

```python
for chunk in graph.stream(
    {"foo": "foo"},
    subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. 设置 `subgraphs=True` 以流式传输子图的输出。

??? example "从子图流式输出"

    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # 定义子图
    class SubgraphState(TypedDict):
        foo: str
        bar: str
    
    def subgraph_node_1(state: SubgraphState):
        return {"bar": "bar"}
    
    def subgraph_node_2(state: SubgraphState):
        # 注意此节点正在使用仅在子图中可用的状态键 ('bar')
        # 并正在向共享状态键 ('foo') 发送更新
        return {"foo": state["foo"] + state["bar"]}
    
    subgraph_builder = StateGraph(SubgraphState)
    subgraph_builder.add_node(subgraph_node_1)
    subgraph_builder.add_node(subgraph_node_2)
    subgraph_builder.add_edge(START, "subgraph_node_1")
    subgraph_builder.add_edge("subgraph_node_1", "subgraph_node_2")
    subgraph = subgraph_builder.compile()
    
    # 定义父图
    class ParentState(TypedDict):
        foo: str
    
    def node_1(state: ParentState):
        return {"foo": "hi! " + state["foo"]}
    
    builder = StateGraph(ParentState)
    builder.add_node("node_1", node_1)
    builder.add_node("node_2", subgraph)
    builder.add_edge(START, "node_1")
    builder.add_edge("node_1", "node_2")
    graph = builder.compile()

    for chunk in graph.stream(
        {"foo": "foo"},
        stream_mode="updates",
        subgraphs=True, # (1)!
    ):
        print(chunk)
    ```
  
    1. 设置 `subgraphs=True` 以流式传输子图的输出。

    ```
    ((), {'node_1': {'foo': 'hi! foo'}})
    (('node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7',), {'subgraph_node_1': {'bar': 'bar'}})
    (('node_2:e58e5673-a661-ebb0-70d4-e298a7fc28b7',), {'subgraph_node_2': {'foo': 'hi! foobar'}})
    ((), {'node_2': {'foo': 'hi! foobar'}})
    ```