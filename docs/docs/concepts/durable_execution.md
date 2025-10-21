---
search:
  boost: 2
---

# 可靠执行

**可靠执行**（Durable execution）是一种技术，指的是一个进程或工作流在关键节点保存其进度，使其能够暂停，并在稍后精确地从中断处恢复。这在需要[人工介入](./human_in_the_loop.md)的场景中特别有用，用户可以在继续之前检查、验证或修改流程；同时也适用于可能会遇到中断或错误的长时间运行的任务（例如，长时间等待 LLM 调用超时）。通过保存已完成的工作，可靠执行使流程无需重新处理之前的步骤即可恢复——即使在长时间延迟之后（例如，一周后）。

LangGraph 内置的[持久化](./persistence.md)层为工作流提供了可靠执行的能力，确保每个执行步骤的状态都已保存到持久存储中。无论工作流是因系统故障还是[人工介入](./human_in_the_loop.md)而中断——它都可以从最后一个记录的状态恢复。

!!! tip

    如果您将 LangGraph 与检查点库（checkpointer）一起使用，那么您已经启用了可靠执行。您可以随时暂停和恢复工作流，即使在中断或失败之后。
    为了充分利用可靠执行，请确保您的工作流设计是[确定性的](#determinism-and-consistent-replay)和[幂等的](#determinism-and-consistent-replay)，并将任何副作用或非确定性操作封装在[任务（tasks）](./functional_api.md#task)中。您可以在 [StateGraph (Graph API)](./low_level.md) 和 [Functional API](./functional_api.md) 中使用[任务（tasks）](./functional_api.md#task)。

## 要求

要在 LangGraph 中利用可靠执行，您需要：

1.  通过指定一个将保存工作流进度的[检查点库（checkpointer）](./persistence.md#checkpointer-libraries)来启用工作流的[持久化](./persistence.md)。
2.  在执行工作流时指定一个[线程标识符（thread identifier）](./persistence.md#threads)。这将跟踪工作流特定实例的执行历史。

:::python

3.  将任何非确定性操作（例如，随机数生成）或具有副作用的操作（例如，文件写入、API 调用）封装在 @[tasks][task] 中，以确保在恢复工作流时，这些操作不会为特定运行而重复执行，而是从持久化层检索其结果。有关更多信息，请参阅[确定性和一致性重放](#determinism-and-consistent-replay)。

:::

:::js

3.  将任何非确定性操作（例如，随机数生成）或具有副作用的操作（例如，文件写入、API 调用）封装在 @[tasks][task] 中，以确保在恢复工作流时，这些操作不会为特定运行而重复执行，而是从持久化层检索其结果。有关更多信息，请参阅[确定性和一致性重放](#determinism-and-consistent-replay)。

:::

## 确定性和一致性重放

当您恢复工作流运行时，代码**不会**从停止执行的**同一行代码**开始；相反，它会识别一个合适的[恢复点](#starting-points-for-resuming-workflows)来继续执行。这意味着工作流将从[恢复点](#starting-points-for-resuming-workflows)开始重放所有步骤，直到到达停止的点。

因此，在编写用于可靠执行的工作流时，您必须将任何非确定性操作（例如，随机数生成）和任何具有副作用的操作（例如，文件写入、API 调用）封装在[任务（tasks）](./functional_api.md#task)或[节点（nodes）](./low_level.md#nodes)中。

为确保您的工作流是确定性的并且可以一致地重放，请遵循以下指南：

-   **避免重复工作**：如果一个[节点（node）](./low_level.md#nodes)包含多个具有副作用的操作（例如，日志记录、文件写入或网络调用），请将每个操作封装在单独的**任务（task**）中。这样可以确保在恢复工作流时，这些操作不会重复执行，而是从持久化层检索其结果。
-   **封装非确定性操作**：将任何可能产生非确定性结果的代码（例如，随机数生成）封装在**任务（tasks**）或**节点（nodes）**中。这样可以确保在恢复时，工作流遵循完全相同的记录步骤序列并具有相同的输出。
-   **使用幂等操作**：尽可能确保副作用（例如，API 调用、文件写入）是幂等的。这意味着如果一个操作在工作流失败后被重试，它将产生与第一次执行相同的影响。这对于导致数据写入的操作尤其重要。如果一个**任务（task**）开始执行但未能成功完成，工作流的恢复将重新运行该**任务（task**），并依赖记录的输出以保持一致性。使用幂等键或验证现有结果以避免意外的重复，确保工作流执行平稳且可预测。

:::python
有关一些应避免的常见陷阱的示例，请参阅功能 API 中的[常见陷阱](./functional_api.md#common-pitfalls)部分，其中展示了
如何使用**任务（tasks**）来构建代码以避免这些问题。同样的原则也适用于 @[StateGraph (Graph API)][StateGraph]。
:::

:::js
有关一些应避免的常见陷阱的示例，请参阅功能 API 中的[常见陷阱](./functional_api.md#common-pitfalls)部分，其中展示了
如何使用**任务（tasks**）来构建代码以避免这些问题。同样的原则也适用于 @[StateGraph (Graph API)][StateGraph]。
:::

## 可靠性模式

LangGraph 支持三种可靠性模式，允许您根据应用程序的要求在性能和数据一致性之间取得平衡。可靠性模式从低到高依次为：

-   [`"exit"`](#exit)
-   [`"async"`](#async)
-   [`"sync"`](#sync)

更高的可靠性模式会增加工作流执行的开销。

!!! version-added "Added in version 0.6.0"

    使用 `durability` 参数而不是 `checkpoint_during`（v0.6.0 已弃用）来管理持久化策略：
    
    * `durability="async"` 替换 `checkpoint_during=True`
    * `durability="exit"` 替换 `checkpoint_during=False`
    
    用于管理持久化策略，映射关系如下：

    * `checkpoint_during=True` -> `durability="async"`
    * `checkpoint_during=False` -> `durability="exit"`

### `"exit"`

更改仅在图执行完成（成功或出错）时持久化。这为长时间运行的图提供了最佳性能，但意味着中间状态未保存，因此您无法从中恢复执行中的失败或中断图的执行。

### `"async"`

更改在下一步执行期间异步持久化。这提供了良好的性能和可靠性，但如果进程在执行期间崩溃，检查点可能无法写入，存在小风险。

### `"sync"`

更改在下一步开始之前同步持久化。这确保了每个检查点都能在继续执行之前写入，从而在付出一些性能开销的代价下提供高可靠性。

您可以在调用任何图执行方法时指定可靠性模式：

:::python

```python
graph.stream(
    {"input": "test"}, 
    durability="sync"
)
```

:::

## 在节点中使用任务

如果一个[节点（node）](./low_level.md#nodes)包含多个操作，那么将每个操作转换为一个**任务（task**）可能会比重构操作到单独的节点更容易。

:::python
=== "Original"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END
    import requests

    # Define a TypedDict to represent the state
    class State(TypedDict):
        url: str
        result: NotRequired[str]

    def call_api(state: State):
        """Example node that makes an API request."""
        # highlight-next-line
        result = requests.get(state['url']).text[:100]  # Side-effect
        return {
            "result": result
        }

    # Create a StateGraph builder and add a node for the call_api function
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # Connect the start and end nodes to the call_api node
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # Specify a checkpointer
    checkpointer = InMemorySaver()

    # Compile the graph with the checkpointer
    graph = builder.compile(checkpointer=checkpointer)

    # Define a config with a thread ID.
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # Invoke the graph
    graph.invoke({"url": "https://www.example.com"}, config)
    ```

=== "With task"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.func import task
    from langgraph.graph import StateGraph, START, END
    import requests

    # Define a TypedDict to represent the state
    class State(TypedDict):
        urls: list[str]
        result: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """Make a request."""
        # highlight-next-line
        return requests.get(url).text[:100]

    def call_api(state: State):
        """Example node that makes an API request."""
        # highlight-next-line
        requests = [_make_request(url) for url in state['urls']]
        results = [request.result() for request in requests]
        return {
            "results": results
        }

    # Create a StateGraph builder and add a node for the call_api function
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # Connect the start and end nodes to the call_api node
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # Specify a checkpointer
    checkpointer = InMemorySaver()

    # Compile the graph with the checkpointer
    graph = builder.compile(checkpointer=checkpointer)

    # Define a config with a thread ID.
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # Invoke the graph
    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```

:::

:::js
=== "Original"

    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    // Define a Zod schema to represent the state
    const State = z.object({
      url: z.string(),
      result: z.string().optional(),
    });

    const callApi = async (state: z.infer<typeof State>) => {
      // highlight-next-line
      const response = await fetch(state.url);
      const text = await response.text();
      const result = text.slice(0, 100); // Side-effect
      return {
        result,
      };
    };

    // Create a StateGraph builder and add a node for the callApi function
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // Specify a checkpointer
    const checkpointer = new MemorySaver();

    // Compile the graph with the checkpointer
    const graph = builder.compile({ checkpointer });

    // Define a config with a thread ID.
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // Invoke the graph
    await graph.invoke({ url: "https://www.example.com" }, config);
    ```

=== "With task"

    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { MemorySaver } from "@langchain/langgraph";
    import { task } from "@langchain/langgraph";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    // Define a Zod schema to represent the state
    const State = z.object({
      urls: z.array(z.string()),
      results: z.array(z.string()).optional(),
    });

    const makeRequest = task("makeRequest", async (url: string) => {
      // highlight-next-line
      const response = await fetch(url);
      const text = await response.text();
      return text.slice(0, 100);
    });

    const callApi = async (state: z.infer<typeof State>) => {
      // highlight-next-line
      const requests = state.urls.map((url) => makeRequest(url));
      const results = await Promise.all(requests);
      return {
        results,
      };
    };

    // Create a StateGraph builder and add a node for the callApi function
    const builder = new StateGraph(State)
      .addNode("callApi", callApi)
      .addEdge(START, "callApi")
      .addEdge("callApi", END);

    // Specify a checkpointer
    const checkpointer = new MemorySaver();

    // Compile the graph with the checkpointer
    const graph = builder.compile({ checkpointer });

    // Define a config with a thread ID.
    const threadId = uuidv4();
    const config = { configurable: { thread_id: threadId } };

    // Invoke the graph
    await graph.invoke({ urls: ["https://www.example.com"] }, config);
    ```

:::

## 恢复工作流

启用工作流的可靠执行后，您可以恢复以下场景的执行：

:::python

-   **暂停和恢复工作流**：使用 @[interrupt][interrupt] 函数在特定点暂停工作流，并使用 @[Command] 原语通过更新的状态恢复它。有关更多详细信息，请参阅[**人工介入**](./human_in_the_loop.md)。
-   **从失败中恢复**：在发生异常（例如，LLM 服务商中断）后，从上一个成功检查点自动恢复工作流。这涉及使用相同的线程标识符执行工作流，并为其提供 `None` 作为输入值（请参阅功能 API 中此[示例](../how-tos/use-functional-api.md#resuming-after-an-error)）。

  :::

:::js

-   **暂停和恢复工作流**：使用 @[interrupt][interrupt] 函数在特定点暂停工作流，并使用 @[Command] 原语通过更新的状态恢复它。有关更多详细信息，请参阅[**人工介入**](./human_in_the_loop.md)。
-   **从失败中恢复**：在发生异常（例如，LLM 服务商中断）后，从上一个成功检查点自动恢复工作流。这涉及使用相同的线程标识符执行工作流，并为其提供 `null` 作为输入值（请参阅功能 API 中此[示例](../how-tos/use-functional-api.md#resuming-after-an-error)）。

  :::

## 恢复工作流的起点

:::python

-   如果您使用的是 @[StateGraph (Graph API)][StateGraph]，则起点是执行停止的[**节点**](./low_level.md#nodes)的开头。
-   如果您在节点内进行子图调用，起点将是调用了已暂停子图的**父节点**。
    在子图内部，起点将是执行停止的特定[**节点**](./low_level.md#nodes)。
-   如果您使用的是功能 API，起点是执行停止的[**入口点**](./functional_api.md#entrypoint)的开头。

  :::

:::js

-   如果您使用的是 [StateGraph (Graph API)](./low_level.md)，则起点是执行停止的[**节点**](./low_level.md#nodes)的开头。
-   如果您在节点内进行子图调用，起点将是调用了已暂停子图的**父节点**。
    在子图内部，起点将是执行停止的特定[**节点**](./low_level.md#nodes)。
-   如果您使用的是功能 API，起点是执行停止的[**入口点**](./functional_api.md#entrypoint)的开头。

  :::