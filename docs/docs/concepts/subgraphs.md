# 子图

子图是作为另一个图中的节点使用的图——这是应用于 LangGraph 的封装概念。子图允许您构建具有多个组件的复杂系统，而这些组件本身就是图。

![子图](./img/subgraph.png)

使用子图的一些原因包括：

- 构建[多智能体系统](./multi_agent.md)
- 当您想在多个图中重用一组节点时
- 当您希望不同的团队独立处理图的不同部分时，您可以将每个部分定义为一个子图，并且只要尊重子图接口（输入和输出模式），就可以构建父图，而无需了解子图的任何详细信息。

添加子图时的主要问题是父图和子图如何通信，即在图执行期间它们如何相互传递状态。有两种情况：

- 父图和子图在其状态[模式](./low_level.md#state)中具有**共享状态键**。在这种情况下，您可以将子图包含为父图中的一个节点。

```python
from langgraph.graph import StateGraph, MessagesState, START

# 子图

def call_model(state: MessagesState):
    response = model.invoke(state["messages"])
    return {"messages": response}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(call_model)
...
# highlight-next-line
subgraph = subgraph_builder.compile()

# 父图

builder = StateGraph(State)
# highlight-next-line
builder.add_node("subgraph_node", subgraph)
builder.add_edge(START, "subgraph_node")
graph = builder.compile()
...
graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
```

- 父图和子图具有**不同的模式**（在其状态[模式](./low_level.md#state)中没有共享状态键）。在这种情况下，您必须从父级图中的节点内部调用子图：这在父图和子图具有不同的状态模式并且您需要在调用子图之前或之后转换状态时非常有用。

```python
from typing_extensions import TypedDict, Annotated
from langchain_core.messages import AnyMessage
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.graph.message import add_messages

class SubgraphMessagesState(TypedDict):
    # highlight-next-line
    subgraph_messages: Annotated[list[AnyMessage], add_messages]

# 子图

# highlight-next-line
def call_model(state: SubgraphMessagesState):
    response = model.invoke(state["subgraph_messages"])
    return {"subgraph_messages": response}

subgraph_builder = StateGraph(SubgraphMessagesState)
subgraph_builder.add_node("call_model_from_subgraph", call_model)
subgraph_builder.add_edge(START, "call_model_from_subgraph")
...
# highlight-next-line
subgraph = subgraph_builder.compile()

# 父图

def call_subgraph(state: MessagesState):
    response = subgraph.invoke({"subgraph_messages": state["messages"]})
    return {"messages": response["subgraph_messages"]}

builder = StateGraph(State)
# highlight-next-line
builder.add_node("subgraph_node", call_subgraph)
builder.add_edge(START, "subgraph_node")
graph = builder.compile()
...
graph.invoke({"messages": [{"role": "user", "content": "hi!"}]})
```