# 子图

子图（Subgraph）是作为另一个图的节点（node）使用的图（graph）——这是封装（encapsulation）概念在 LangGraph 中的应用。子图允许你构建包含多个组件的复杂系统，而这些组件本身就是图。

![子图](./img/subgraph.png)

使用子图的一些原因包括：

- 构建[多智能体系统](./multi_agent.md)
- 当你想在多个图中复用一组节点时
- 当你希望不同的团队独立开发图的不同部分时，可以将每个部分定义为一个子图，只要子图的接口（输入和输出模式）得到遵循，父图就可以在不了解子图任何细节的情况下进行构建

添加子图时的主要问题是父图和子图如何通信，即在图执行过程中它们如何传递[状态](./low_level.md#state)。有两种情况：

- 父图和子图在其状态[模式（schemas）](./low_level.md#state)中具有**共享的状态键**。在这种情况下，你可以[将子图作为节点包含在父图中](../how-tos/subgraph.ipynb#shared-state-schemas)。

  :::python

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

  :::

  :::js

  ```typescript
  import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";

  // 子图

  const subgraphBuilder = new StateGraph(MessagesZodState).addNode(
    "callModel",
    async (state) => {
      const response = await model.invoke(state.messages);
      return { messages: response };
    }
  );
  // ... 其他节点和边
  // highlight-next-line
  const subgraph = subgraphBuilder.compile();

  // 父图

  const builder = new StateGraph(MessagesZodState)
    // highlight-next-line
    .addNode("subgraphNode", subgraph)
    .addEdge(START, "subgraphNode");
  const graph = builder.compile();
  // ...
  await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
  ```

  :::

- 父图和子图具有**不同的模式**（在其状态[模式（schemas）](./low_level.md#state)中没有共享的状态键）。在这种情况下，你必须[从父图的节点内部调用子图](../how-tos/subgraph.ipynb#different-state-schemas)：这在父图和子图具有不同的状态模式，并且你需要在调用子图之前或之后转换状态时很有用。

  :::python

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

  :::

  :::js

  ```typescript
  import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
  import { z } from "zod";

  const SubgraphState = z.object({
    // highlight-next-line
    subgraphMessages: MessagesZodState.shape.messages,
  });

  // 子图

  const subgraphBuilder = new StateGraph(SubgraphState)
    // highlight-next-line
    .addNode("callModelFromSubgraph", async (state) => {
      const response = await model.invoke(state.subgraphMessages);
      return { subgraphMessages: response };
    })
    .addEdge(START, "callModelFromSubgraph");
  // ...
  // highlight-next-line
  const subgraph = subgraphBuilder.compile();

  // 父图

  const builder = new StateGraph(MessagesZodState)
    // highlight-next-line
    .addNode("subgraphNode", async (state) => {
      const response = await subgraph.invoke({
        subgraphMessages: state.messages,
      });
      return { messages: response.subgraphMessages };
    })
    .addEdge(START, "subgraphNode");
  const graph = builder.compile();
  // ...
  await graph.invoke({ messages: [{ role: "user", content: "hi!" }] });
  ```

  :::