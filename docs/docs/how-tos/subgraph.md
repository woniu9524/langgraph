# 使用子图

本指南解释了使用 [子图](../concepts/subgraphs.md) 的机制。子图的一个常见应用是构建 [多智能体](../concepts/multi_agent.md) 系统。

添加子图时，需要定义父图和子图如何通信：

*   [共享状态模式](#shared-state-schemas) — 父图和子图在其状态 [模式](../concepts/low_level.md#state) 中具有**共享状态键**
*   [不同状态模式](#different-state-schemas) — 父图和子图 [模式](../concepts/low_level.md#state) **没有共享状态键**

## 设置

:::python
```bash
pip install -U langgraph
```
:::

:::js
```bash
npm install @langchain/langgraph
```
:::

!!! tip "为 LangGraph 开发设置 LangSmith"

    注册 [LangSmith](https://smith.langchain.com) 以快速发现问题并改进 LangGraph 项目的性能。LangSmith 允许您使用跟踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用 — 在[此处](https://docs.smith.langchain.com) 阅读有关入门的更多信息。

## 共享状态模式

一种常见的情况是父图和子图通过 [模式](../concepts/low_level.md#state) 中的共享状态键（通道）进行通信。例如，在 [多智能体](../concepts/multi_agent.md) 系统中，智能体通常通过共享的 [消息](https://langchain-ai.github.io/langgraph/concepts/low_level.md#why-use-messages) 键进行通信。

如果您的子图与父图共享状态键，您可以按照以下步骤将其添加到图中：

:::python
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
:::

:::js
1. 定义子图工作流（下方示例中的 `subgraphBuilder`）并编译它
2. 在定义父图工作流时，将编译后的子图传递给 `.addNode` 方法

```typescript
import { StateGraph, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
});

// 子图
const subgraphBuilder = new StateGraph(State)
  .addNode("subgraphNode1", (state) => {
    return { foo: "hi! " + state.foo };
  })
  .addEdge(START, "subgraphNode1");

const subgraph = subgraphBuilder.compile();

// 父图
const builder = new StateGraph(State)
  .addNode("node1", subgraph)
  .addEdge(START, "node1");

const graph = builder.compile();
```
:::

??? example "完整示例：共享状态模式"

    :::python
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
        # 注意：此节点使用的是子图中唯一存在的状态键 ('bar')
        # 并向共享状态键 ('foo') 发送更新
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
    2. 此键仅对 `SubgraphState` 可见，父图无法访问
    
    ```
    {'node_1': {'foo': 'hi! foo'}}
    {'node_2': {'foo': 'hi! foobar'}}
    ```
    :::

    :::js
    ```typescript
    import { StateGraph, START } from "@langchain/langgraph";
    import { z } from "zod";

    // 定义子图
    const SubgraphState = z.object({
      foo: z.string(),  // (1)! 
      bar: z.string(),  // (2)!
    });
    
    const subgraphBuilder = new StateGraph(SubgraphState)
      .addNode("subgraphNode1", (state) => {
        return { bar: "bar" };
      })
      .addNode("subgraphNode2", (state) => {
        // 注意：此节点使用的是子图中唯一存在的状态键 ('bar')
        // 并向共享状态键 ('foo') 发送更新
        return { foo: state.foo + state.bar };
      })
      .addEdge(START, "subgraphNode1")
      .addEdge("subgraphNode1", "subgraphNode2");
    
    const subgraph = subgraphBuilder.compile();
    
    // 定义父图
    const ParentState = z.object({
      foo: z.string(),
    });
    
    const builder = new StateGraph(ParentState)
      .addNode("node1", (state) => {
        return { foo: "hi! " + state.foo };
      })
      .addNode("node2", subgraph)
      .addEdge(START, "node1")
      .addEdge("node1", "node2");
    
    const graph = builder.compile();
    
    for await (const chunk of await graph.stream({ foo: "foo" })) {
      console.log(chunk);
    }
    ```

    3. 此键与父图状态共享
    4. 此键仅对 `SubgraphState` 可见，父图无法访问
    
    ```
    { node1: { foo: 'hi! foo' } }
    { node2: { foo: 'hi! foobar' } }
    ```
    :::

## 不同状态模式

对于更复杂的系统，您可能希望定义具有与父图**完全不同模式**（无共享键）的子图。例如，您可能希望为多智能体系统中的每个智能体维护私有消息历史记录。

如果您的应用程序是这种情况，您需要定义一个**调用子图的节点函数**。此函数需要在调用子图之前将输入（父）状态转换为子图状态，并在返回节点的状态更新之前将结果转换回父状态。

:::python
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
:::

:::js
```typescript
import { StateGraph, START } from "@langchain/langgraph";
import { z } from "zod";

const SubgraphState = z.object({
  bar: z.string(),
});

// 子图
const subgraphBuilder = new StateGraph(SubgraphState)
  .addNode("subgraphNode1", (state) => {
    return { bar: "hi! " + state.bar };
  })
  .addEdge(START, "subgraphNode1");

const subgraph = subgraphBuilder.compile();

// 父图
const State = z.object({
  foo: z.string(),
});

const builder = new StateGraph(State)
  .addNode("node1", async (state) => {
    const subgraphOutput = await subgraph.invoke({ bar: state.foo }); // (1)!
    return { foo: subgraphOutput.bar }; // (2)!
  })
  .addEdge(START, "node1");

const graph = builder.compile();
```

1. 将状态转换为子图状态
2. 将响应转换回父状态
:::

??? example "完整示例：不同状态模式"

    :::python
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph.state import StateGraph, START

    # 定义子图
    class SubgraphState(TypedDict):
        # 注意：这些键均未与父图状态共享
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
    :::

    :::js
    ```typescript
    import { StateGraph, START } from "@langchain/langgraph";
    import { z } from "zod";

    // 定义子图
    const SubgraphState = z.object({
      // 注意：这些键均未与父图状态共享
      bar: z.string(),
      baz: z.string(),
    });
    
    const subgraphBuilder = new StateGraph(SubgraphState)
      .addNode("subgraphNode1", (state) => {
        return { baz: "baz" };
      })
      .addNode("subgraphNode2", (state) => {
        return { bar: state.bar + state.baz };
      })
      .addEdge(START, "subgraphNode1")
      .addEdge("subgraphNode1", "subgraphNode2");
    
    const subgraph = subgraphBuilder.compile();
    
    // 定义父图
    const ParentState = z.object({
      foo: z.string(),
    });
    
    const builder = new StateGraph(ParentState)
      .addNode("node1", (state) => {
        return { foo: "hi! " + state.foo };
      })
      .addNode("node2", async (state) => {
        const response = await subgraph.invoke({ bar: state.foo }); // (1)!
        return { foo: response.bar }; // (2)!
      })
      .addEdge(START, "node1")
      .addEdge("node1", "node2");
    
    const graph = builder.compile();
    
    for await (const chunk of await graph.stream(
      { foo: "foo" }, 
      { subgraphs: true }
    )) {
      console.log(chunk);
    }
    ```

    3. 将状态转换为子图状态
    4. 将响应转换回父状态

    ```
    [[], { node1: { foo: 'hi! foo' } }]
    [['node2:9c36dd0f-151a-cb42-cbad-fa2f851f9ab7'], { subgraphNode1: { baz: 'baz' } }]
    [['node2:9c36dd0f-151a-cb42-cbad-fa2f851f9ab7'], { subgraphNode2: { bar: 'hi! foobaz' } }]
    [[], { node2: { foo: 'hi! foobaz' } }]
    ```
    :::

??? example "完整示例：不同状态模式（两层子图）"

    这是两层子图的示例：父图 -> 子图 -> 孙子图。

    :::python
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
    3. 这里传递的是一个函数，而非仅编译后的图 (`grandchild_graph`)
    4. 我们正在将状态从父图状态通道 (`my_key`) 转换为子图状态通道 (`my_child_key`)
    5. 我们正在将状态从子图状态通道 (`my_child_key`) 转换回父图状态通道 (`my_key`)
    6. 这里传递的是一个函数，而非仅编译后的图 (`child_graph`)

    ```
    ((), {'parent_1': {'my_key': 'hi Bob'}})
    (('child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b', 'child_1:781bb3b1-3971-84ce-810b-acf819a03f9c'), {'grandchild_1': {'my_grandchild_key': 'hi Bob, how are you'}})
    (('child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b',), {'child_1': {'my_child_key': 'hi Bob, how are you today?'}})
    ((), {'child': {'my_key': 'hi Bob, how are you today?'}})
    ((), {'parent_2': {'my_key': 'hi Bob, how are you today? bye!'}})
    ```
    :::

    :::js
    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { z } from "zod";

    // 孙子图
    const GrandChildState = z.object({
      myGrandchildKey: z.string(),
    });
    
    const grandchild = new StateGraph(GrandChildState)
      .addNode("grandchild1", (state) => {
        // 注意：在此处无法访问子图或父图的键
        return { myGrandchildKey: state.myGrandchildKey + ", how are you" };
      })
      .addEdge(START, "grandchild1")
      .addEdge("grandchild1", END);
    
    const grandchildGraph = grandchild.compile();
    
    // 子图
    const ChildState = z.object({
      myChildKey: z.string(),
    });
    
    const child = new StateGraph(ChildState)
      .addNode("child1", async (state) => {
        // 注意：在此处无法访问父图或孙子图的键
        const grandchildGraphInput = { myGrandchildKey: state.myChildKey }; // (1)!
        const grandchildGraphOutput = await grandchildGraph.invoke(grandchildGraphInput);
        return { myChildKey: grandchildGraphOutput.myGrandchildKey + " today?" }; // (2)!
      }) // (3)!
      .addEdge(START, "child1")
      .addEdge("child1", END);
    
    const childGraph = child.compile();
    
    // 父图
    const ParentState = z.object({
      myKey: z.string(),
    });
    
    const parent = new StateGraph(ParentState)
      .addNode("parent1", (state) => {
        // 注意：在此处无法访问子图或孙子图的键
        return { myKey: "hi " + state.myKey };
      })
      .addNode("child", async (state) => {
        const childGraphInput = { myChildKey: state.myKey }; // (4)!
        const childGraphOutput = await childGraph.invoke(childGraphInput);
        return { myKey: childGraphOutput.myChildKey }; // (5)!
      }) // (6)!
      .addNode("parent2", (state) => {
        return { myKey: state.myKey + " bye!" };
      })
      .addEdge(START, "parent1")
      .addEdge("parent1", "child")
      .addEdge("child", "parent2")
      .addEdge("parent2", END);
    
    const parentGraph = parent.compile();
    
    for await (const chunk of await parentGraph.stream(
      { myKey: "Bob" }, 
      { subgraphs: true }
    )) {
      console.log(chunk);
    }
    ```

    7. 我们正在将状态从子图状态通道 (`myChildKey`) 转换为孙子图状态通道 (`myGrandchildKey`)
    8. 我们正在将状态从孙子图状态通道 (`myGrandchildKey`) 转换回子图状态通道 (`myChildKey`)
    9. 这里传递的是一个函数，而非仅编译后的图 (`grandchildGraph`)
    10. 我们正在将状态从父图状态通道 (`myKey`) 转换为子图状态通道 (`myChildKey`)
    11. 我们正在将状态从子图状态通道 (`myChildKey`) 转换回父图状态通道 (`myKey`)
    12. 这里传递的是一个函数，而非仅编译后的图 (`childGraph`)

    ```
    [[], { parent1: { myKey: 'hi Bob' } }]
    [['child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b', 'child1:781bb3b1-3971-84ce-810b-acf819a03f9c'], { grandchild1: { myGrandchildKey: 'hi Bob, how are you' } }]
    [['child:2e26e9ce-602f-862c-aa66-1ea5a4655e3b'], { child1: { myChildKey: 'hi Bob, how are you today?' } }]
    [[], { child: { myKey: 'hi Bob, how are you today?' } }]
    [[], { parent2: { myKey: 'hi Bob, how are you today? bye!' } }]
    ```
    :::

## 添加持久化

您只需要在**编译父图时提供 checkpointer**。LangGraph 会自动将 checkpointer 传播给子图。

:::python
```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import MemorySaver
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

checkpointer = MemorySaver()
graph = builder.compile(checkpointer=checkpointer)
```    
:::

:::js
```typescript
import { StateGraph, START, MemorySaver } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
});

// 子图
const subgraphBuilder = new StateGraph(State)
  .addNode("subgraphNode1", (state) => {
    return { foo: state.foo + "bar" };
  })
  .addEdge(START, "subgraphNode1");

const subgraph = subgraphBuilder.compile();

// 父图
const builder = new StateGraph(State)
  .addNode("node1", subgraph)
  .addEdge(START, "node1");

const checkpointer = new MemorySaver();
const graph = builder.compile({ checkpointer });
```    
:::

如果您希望子图**拥有自己的内存**，可以编译子图时设置相应的 checkpointer 选项。这在 [多智能体](../concepts/multi_agent.md)系统中非常有用，如果您希望智能体能够跟踪其内部消息历史记录：

:::python
```python
subgraph_builder = StateGraph(...)
subgraph = subgraph_builder.compile(checkpointer=True)
```
:::

:::js
```typescript
const subgraphBuilder = new StateGraph(...)
const subgraph = subgraphBuilder.compile({ checkpointer: true });
```
:::

## 查看子图状态

启用 [持久化](../concepts/persistence.md) 后，您可以通过相应的方法 [检查图状态](../concepts/persistence.md#checkpoints)（检查点）。要查看子图状态，可以使用 subgraphs 选项。

:::python
您可以通过 `graph.get_state(config)` 检查图状态。要查看子图状态，可以使用 `graph.get_state(config, subgraphs=True)`。
:::

:::js
您可以通过 `graph.getState(config)` 检查图状态。要查看子图状态，可以使用 `graph.getState(config, { subgraphs: true })`。
:::

!!! important "仅在中断时可用"

    子图状态只能在**子图被中断时**查看。一旦您恢复图，将无法访问子图状态。

??? example "查看中断的子图状态"

    :::python
    ```python
    from langgraph.graph import START, StateGraph
    from langgraph.checkpoint.memory import MemorySaver
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
    
    checkpointer = MemorySaver()
    graph = builder.compile(checkpointer=checkpointer)
    
    config = {"configurable": {"thread_id": "1"}}
    
    graph.invoke({"foo": ""}, config)
    parent_state = graph.get_state(config)
    subgraph_state = graph.get_state(config, subgraphs=True).tasks[0].state  # (1)!
    
    # 恢复子图
    graph.invoke(Command(resume="bar"), config)
    ```
    
    1. 这种情况仅在子图被中断时可用。一旦您恢复图，将无法访问子图状态。
    :::

    :::js
    ```typescript
    import { StateGraph, START, MemorySaver, interrupt, Command } from "@langchain/langgraph";
    import { z } from "zod";
    
    const State = z.object({
      foo: z.string(),
    });
    
    // 子图
    const subgraphBuilder = new StateGraph(State)
      .addNode("subgraphNode1", (state) => {
        const value = interrupt("Provide value:");
        return { foo: state.foo + value };
      })
      .addEdge(START, "subgraphNode1");
    
    const subgraph = subgraphBuilder.compile();
    
    // 父图
    const builder = new StateGraph(State)
      .addNode("node1", subgraph)
      .addEdge(START, "node1");
    
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });
    
    const config = { configurable: { thread_id: "1" } };
    
    await graph.invoke({ foo: "" }, config);
    const parentState = await graph.getState(config);
    const subgraphState = (await graph.getState(config, { subgraphs: true })).tasks[0].state; // (1)!
    
    // 恢复子图
    await graph.invoke(new Command({ resume: "bar" }), config);
    ```
    
    2. 这种情况仅在子图被中断时可用。一旦您恢复图，将无法访问子图状态。
    :::

## 流式传输子图输出

要将子图的输出包含在流式传输的输出中，可以在父图的 `stream` 方法中设置 `subgraphs` 选项。这将流式传输父图和任何子图的输出。

:::python
```python
for chunk in graph.stream(
    {"foo": "foo"},
    subgraphs=True, # (1)!
    stream_mode="updates",
):
    print(chunk)
```

1. 设置 `subgraphs=True` 以流式传输子图的输出。
:::

:::js
```typescript
for await (const chunk of await graph.stream(
  { foo: "foo" },
  {
    subgraphs: true, // (1)!
    streamMode: "updates",
  }
)) {
  console.log(chunk);
}
```

1. 设置 `subgraphs: true` 以流式传输子图的输出。
:::

??? example "从子图流式传输"

    :::python
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
        # 注意：此节点使用的是子图中唯一存在的状态键 ('bar')
        # 并向共享状态键 ('foo') 发送更新
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
    :::

    :::js
    ```typescript
    import { StateGraph, START } from "@langchain/langgraph";
    import { z } from "zod";

    // 定义子图
    const SubgraphState = z.object({
      foo: z.string(),
      bar: z.string(),
    });
    
    const subgraphBuilder = new StateGraph(SubgraphState)
      .addNode("subgraphNode1", (state) => {
        return { bar: "bar" };
      })
      .addNode("subgraphNode2", (state) => {
        // 注意：此节点使用的是子图中唯一存在的状态键 ('bar')
        // 并向共享状态键 ('foo') 发送更新
        return { foo: state.foo + state.bar };
      })
      .addEdge(START, "subgraphNode1")
      .addEdge("subgraphNode1", "subgraphNode2");
    
    const subgraph = subgraphBuilder.compile();
    
    // 定义父图
    const ParentState = z.object({
      foo: z.string(),
    });
    
    const builder = new StateGraph(ParentState)
      .addNode("node1", (state) => {
        return { foo: "hi! " + state.foo };
      })
      .addNode("node2", subgraph)
      .addEdge(START, "node1")
      .addEdge("node1", "node2");
    
    const graph = builder.compile();

    for await (const chunk of await graph.stream(
      { foo: "foo" },
      {
        streamMode: "updates",
        subgraphs: true, // (1)!
      }
    )) {
      console.log(chunk);
    }
    ```
  
    2. 设置 `subgraphs: true` 以流式传输子图的输出。

    ```
    [[], { node1: { foo: 'hi! foo' } }]
    [['node2:e58e5673-a661-ebb0-70d4-e298a7fc28b7'], { subgraphNode1: { bar: 'bar' } }]
    [['node2:e58e5673-a661-ebb0-70d4-e298a7fc28b7'], { subgraphNode2: { foo: 'hi! foobar' } }]
    [[], { node2: { foo: 'hi! foobar' } }]
    ```
    :::