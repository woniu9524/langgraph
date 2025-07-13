---
search:
  boost: 2
---

# LangGraph 运行时

[Pregel][langgraph.pregel.Pregel] 实现了 LangGraph 的运行时，负责管理 LangGraph 应用程序的执行。

编译一个 [StateGraph][langgraph.graph.StateGraph] 或创建一个 [入口点][langgraph.func.entrypoint]会生成一个可以被调用的 [Pregel][langgraph.pregel.Pregel] 实例。

本指南将从高层解释这个运行时，并提供直接使用 Pregel 实现应用程序的说明。

> **注意：** [Pregel][langgraph.pregel.Pregel] 运行时之所以命名为 [Google 的 Pregel 算法](https://research.google/pubs/pub37252/)，是因为该算法描述了一种使用图进行大规模并行计算的高效方法。

## 概述

在 LangGraph 中，Pregel 将 [**Actor**](https://en.wikipedia.org/wiki/Actor_model) 和 **Channel** 整合到一个单一的应用程序中。**Actor** 从 Channel 读取数据并向 Channel 写入数据。Pregel 将应用程序的执行组织成多个步骤，遵循 **Pregel 算法**/**批量同步并行**（Bulk Synchronous Parallel）模型。

每个步骤包含三个阶段：

- **规划 (Plan)**：确定此步骤中要执行的 **Actor**。例如，在第一步，选择订阅特殊 **输入 (input)** Channel 的 **Actor**；在后续步骤中，选择在前一 步更新的 Channel 上订阅的 **Actor**。
- **执行 (Execution)**：并行执行所有选定的 **Actor**，直到所有 Actor 完成，或其中一个失败，或达到超时时间。在此阶段，Channel 的更新对 Actor 是不可见的，直到下一个步骤。
- **更新 (Update)**：用此步骤中 **Actor** 写入的值来更新 Channel。

重复此过程，直到没有 **Actor** 被选中执行，或者达到最大步数。

## Actor

Actor 是一个 `PregelNode`。它订阅 Channel，从 Channel 读取数据并写入数据。可以将其视为 Pregel 算法中的一个 **Actor**。`PregelNodes` 实现 LangChain 的 Runnable 接口。

## Channel

Channel 用于 Actor（PregelNodes）之间的通信。每个 Channel 都有一个值类型、一个更新类型和一个更新函数——该函数接收一系列更新并修改存储的值。Channel 可用于将数据从一个链发送到另一个链，或将数据从一个链发送到未来步骤的自身。LangGraph 提供了一些内置的 Channel：

- [LastValue][langgraph.channels.LastValue]：默认 Channel，存储发送到 Channel 的最后一个值，适用于输入和输出值，或用于将数据从一个步骤发送到下一个步骤。
- [Topic][langgraph.channels.Topic]：一个可配置的发布/订阅（PubSub）主题，适用于在 **Actor** 之间发送多个值，或用于累积输出。可以配置为去重值或在多个步骤中累积值。
- [BinaryOperatorAggregate][langgraph.channels.BinaryOperatorAggregate]：存储一个持久值，通过将一个二元运算符应用于当前值和发送到 Channel 的每个更新来更新，适用于计算跨多个步骤的聚合；例如，`total = BinaryOperatorAggregate(int, operator.add)`

## 示例

虽然大多数用户会通过 [StateGraph][langgraph.graph.StateGraph] API 或
[入口点][langgraph.func.entrypoint] 装饰器与 Pregel 交互，但也可以直接与 Pregel 交互。

以下是一些不同的示例，让您了解 Pregel API。

=== "单个节点"

    ```python

    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    app = Pregel(
        nodes={"node1": node1},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo'}
    ```

=== "多个节点"

    ```python
    from langgraph.channels import LastValue, EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )


    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": LastValue(str),
            "c": EphemeralValue(str),
        },
        input_channels=["a"],
        output_channels=["b", "c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```con
    {'b': 'foofoo', 'c': 'foofoofoofoo'}
    ```

=== "Topic"

    ```python
    from langgraph.channels import EphemeralValue, Topic
    from langgraph.pregel import Pregel, NodeBuilder

    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_to("b")
        .do(lambda x: x["b"] + x["b"])
        .write_to("c")
    )

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": Topic(str, accumulate=True),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

    ```pycon
    {'c': ['foofoo', 'foofoofoofoo']}
    ```

=== "BinaryOperatorAggregate"

    此示例演示如何使用 BinaryOperatorAggregate Channel 实现一个reducer。

    ```python
    from langgraph.channels import EphemeralValue, BinaryOperatorAggregate
    from langgraph.pregel import Pregel, NodeBuilder


    node1 = (
        NodeBuilder().subscribe_only("a")
        .do(lambda x: x + x)
        .write_to("b", "c")
    )

    node2 = (
        NodeBuilder().subscribe_only("b")
        .do(lambda x: x + x)
        .write_to("c")
    )

    def reducer(current, update):
        if current:
            return current + " | " + update
        else:
            return update

    app = Pregel(
        nodes={"node1": node1, "node2": node2},
        channels={
            "a": EphemeralValue(str),
            "b": EphemeralValue(str),
            "c": BinaryOperatorAggregate(str, operator=reducer),
        },
        input_channels=["a"],
        output_channels=["c"],
    )

    app.invoke({"a": "foo"})
    ```

=== "循环"

    此示例演示如何通过让一个链写入其订阅的 Channel 来引入图中的循环。执行将持续进行，直到向 Channel 写入 None 值。

    ```python
    from langgraph.channels import EphemeralValue
    from langgraph.pregel import Pregel, NodeBuilder, ChannelWriteEntry

    example_node = (
        NodeBuilder().subscribe_only("value")
        .do(lambda x: x + x if len(x) < 10 else None)
        .write_to(ChannelWriteEntry("value", skip_none=True))
    )

    app = Pregel(
        nodes={"example_node": example_node},
        channels={
            "value": EphemeralValue(str),
        },
        input_channels=["value"],
        output_channels=["value"],
    )

    app.invoke({"value": "a"})
    ```

    ```pycon
    {'value': 'aaaaaaaaaaaaaaaa'}
    ```

## 高层 API

LangGraph 提供了两个高层 API 来创建 Pregel 应用程序：[StateGraph (Graph API)](./low_level.md) 和 [函数式 API](functional_api.md)。

=== "StateGraph (Graph API)"

    [StateGraph (Graph API)][langgraph.graph.StateGraph] 是一个更高级的抽象，它简化了 Pregel 应用程序的创建。它允许您定义一个由节点和边组成的图。当您编译图时，StateGraph API 会自动为您创建 Pregel 应用程序。

    ```python
    from typing import TypedDict, Optional

    from langgraph.constants import START
    from langgraph.graph import StateGraph

    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]

    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    def score_essay(essay: Essay):
        return {
            "score": 10
        }

    builder = StateGraph(Essay)
    builder.add_node(write_essay)
    builder.add_node(score_essay)
    builder.add_edge(START, "write_essay")

    # Compile the graph.
    # This will return a Pregel instance.
    graph = builder.compile()
    ```

    编译后的 Pregel 实例将与节点和 Channel 列表相关联。您可以通过打印它们来查看节点和 Channel。

    ```python
    print(graph.nodes)
    ```

    您会看到类似这样的内容：

    ```pycon
    {'__start__': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1810>,
     'write_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba14d0>,
     'score_essay': <langgraph.pregel.read.PregelNode at 0x7d05e3ba1710>}
    ```

    ```python
    print(graph.channels)
    ```

    您应该会看到类似这样的内容

    ```pycon
    {'topic': <langgraph.channels.last_value.LastValue at 0x7d05e3294d80>,
     'content': <langgraph.channels.last_value.LastValue at 0x7d05e3295040>,
     'score': <langgraph.channels.last_value.LastValue at 0x7d05e3295980>,
     '__start__': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3297e00>,
     'write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32960c0>,
     'score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ab80>,
     'branch:__start__:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e32941c0>,
     'branch:__start__:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d88800>,
     'branch:write_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e3295ec0>,
     'branch:write_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8ac00>,
     'branch:score_essay:__self__:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d89700>,
     'branch:score_essay:__self__:score_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b400>,
     'start:write_essay': <langgraph.channels.ephemeral_value.EphemeralValue at 0x7d05e2d8b280>}
    ```

=== "函数式 API"

    在 [函数式 API](functional_api.md) 中，您可以使用 [`entrypoint`][langgraph.func.entrypoint] 来创建
    一个 Pregel 应用程序。`entrypoint` 装饰器允许您定义一个接受输入并返回输出的函数。

    ```python
    from typing import TypedDict, Optional

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import entrypoint

    class Essay(TypedDict):
        topic: str
        content: Optional[str]
        score: Optional[float]


    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def write_essay(essay: Essay):
        return {
            "content": f"Essay about {essay['topic']}",
        }

    print("Nodes: ")
    print(write_essay.nodes)
    print("Channels: ")
    print(write_essay.channels)
    ```

    ```pycon
    Nodes:
    {'write_essay': <langgraph.pregel.read.PregelNode object at 0x7d05e2f9aad0>}
    Channels:
    {'__start__': <langgraph.channels.ephemeral_value.EphemeralValue object at 0x7d05e2c906c0>, '__end__': <langgraph.channels.last_value.LastValue object at 0x7d05e2c90c40>, '__previous__': <langgraph.channels.last_value.LastValue object at 0x7d05e1007280>}
    ```