---
search:
  boost: 2
---

# 可靠执行

**可靠执行** 是一种技术，其中进程或工作流在关键点保存其进度，允许它暂停并稍后从中断的地方精确恢复。这在需要[人工干预](./human_in_the_loop.md)的场景中特别有用，用户可以在继续之前检查、验证或修改进程，以及在可能遇到中断或错误的长时间运行任务中（例如，LLM 调用超时）。通过保存已完成的工作，可靠执行使进程能够在不重新处理先前步骤的情况下恢复——即使经过了很长时间（例如，一周后）。

LangGraph 内置的[持久化](./persistence.md)层为工作流提供了可靠执行，确保每个执行步骤的状态都已保存到持久化存储中。此功能保证，如果工作流中断——无论是由于系统故障还是[人工干预](./human_in_the_loop.md)交互——它都可以从最后一个记录的状态恢复。

!!! tip

    如果您将 LangGraph 与检查点一起使用，则已启用了可靠执行。您可以随时暂停和恢复工作流，即使在中断或失败之后。
    为了充分利用可靠执行，请确保您的工作流设计是[确定性的](#determinism-and-consistent-replay)和[幂等的](#determinism-and-consistent-replay)，并将任何副作用或非确定性操作包装在[任务](./functional_api.md#task)中。您可以从[StateGraph (Graph API)](./low_level.md) 和[Functional API](./functional_api.md) 使用[任务](./functional_api.md#task)。

## 要求

要利用 LangGraph 中的可靠执行，您需要：

1.  通过指定一个将保存工作流进度的[检查点](./persistence.md#checkpointer-libraries)来启用工作流中的[持久化](./persistence.md)。
2.  在执行工作流时指定一个[线程标识符](./persistence.md#threads)。这将跟踪特定工作流实例的执行历史。
3.  将任何非确定性操作（例如，随机数生成）或具有副作用的操作（例如，文件写入、API 调用）包装在[任务][langgraph.func.task]中，以确保在恢复工作流时，这些操作不会被重复执行，而是从持久化层检索其结果。更多信息，请参阅[确定性和一致重放](#determinism-and-consistent-replay)。

## 确定性和一致重放

当您恢复工作流运行时，代码**不会**从执行停止的**同一行代码**恢复；相反，它会识别一个合适的[恢复起点](#starting-points-for-resuming-workflows)来接续之前的工作。这意味着工作流将从[起点](#starting-points-for-resuming-workflows)重放所有步骤，直到达到停止的点。

因此，在编写用于可靠执行的工作流时，您必须将任何非确定性操作（例如，随机数生成）和任何具有副作用的操作（例如，文件写入、API 调用）包装在[任务](./functional_api.md#task)或[节点](./low_level.md#nodes)中。

为确保您的工作流是确定性的并且可以一致地重放，请遵循以下指南：

-   **避免重复工作**：如果一个[节点](./low_level.md#nodes)包含多个具有副作用的操作（例如，日志记录、文件写入或网络调用），请将每个操作包装在一个单独的**任务**中。这确保了在恢复工作流时，不会重复执行这些操作，而是从持久化层检索它们的结果。
-   **封装非确定性操作**：将可能产生非确定性结果的代码（例如，随机数生成）包装在**任务**或**节点**中。这确保了在恢复时，工作流遵循确切记录的步骤顺序并产生相同的结果。
-   **使用幂等操作**：如果可能，请确保副作用（例如，API 调用、文件写入）是幂等的。这意味着如果工作流中的失败后重试某个操作，它将产生与第一次执行时相同的影响。这对于导致数据写入的操作尤其重要。如果一个**任务**开始但未能成功完成，工作流的恢复将重新运行该**任务**，依赖于记录的输出以保持一致性。使用幂等键或验证现有结果以避免意外重复，确保工作流执行顺畅且可预测。

有关需要避免的陷阱的示例，请参阅功能 API 中的[常见陷阱](./functional_api.md#common-pitfalls)部分，其中展示了如何使用**任务**来构建代码以避免这些问题。这些原则同样适用于[StateGraph (Graph API)][langgraph.graph.state.StateGraph]。

## 在节点中使用任务

如果一个[节点](./low_level.md#nodes)包含多个操作，您可能会发现将每个操作转换为**任务**比将操作重构为单独的节点更容易。

=== "原始"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import MemorySaver
    from langgraph.graph import StateGraph, START, END
    import requests

    # 定义一个 TypedDict 来表示状态
    class State(TypedDict):
        url: str
        result: NotRequired[str]

    def call_api(state: State):
        """一个执行 API 请求的示例节点。"""
        # highlight-next-line
        result = requests.get(state['url']).text[:100]  # 副作用
        return {
            "result": result
        }

    # 创建一个 StateGraph 构建器并将 call_api 节点添加到其中
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # 将开始和结束节点连接到 call_api 节点
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # 指定一个检查点
    checkpointer = MemorySaver()

    # 使用检查点编译图
    graph = builder.compile(checkpointer=checkpointer)

    # 定义一个带有线程 ID 的配置。
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图
    graph.invoke({"url": "https://www.example.com"}, config)
    ```

=== "使用任务"

    ```python
    from typing import NotRequired
    from typing_extensions import TypedDict
    import uuid

    from langgraph.checkpoint.memory import MemorySaver
    from langgraph.func import task
    from langgraph.graph import StateGraph, START, END
    import requests

    # 定义一个 TypedDict 来表示状态
    class State(TypedDict):
        urls: list[str]
        result: NotRequired[list[str]]


    @task
    def _make_request(url: str):
        """发起请求。"""
        # highlight-next-line
        return requests.get(url).text[:100]

    def call_api(state: State):
        """一个执行 API 请求的示例节点。"""
        # highlight-next-line
        requests = [_make_request(url) for url in state['urls']]
        results = [request.result() for request in requests]
        return {
            "results": results
        }

    # 创建一个 StateGraph 构建器并将 call_api 节点添加到其中
    builder = StateGraph(State)
    builder.add_node("call_api", call_api)

    # 将开始和结束节点连接到 call_api 节点
    builder.add_edge(START, "call_api")
    builder.add_edge("call_api", END)

    # 指定一个检查点
    checkpointer = MemorySaver()

    # 使用检查点编译图
    graph = builder.compile(checkpointer=checkpointer)

    # 定义一个带有线程 ID 的配置。
    thread_id = uuid.uuid4()
    config = {"configurable": {"thread_id": thread_id}}

    # 调用图
    graph.invoke({"urls": ["https://www.example.com"]}, config)
    ```

## 恢复工作流

启用工作流中的可靠执行后，您可以为以下场景恢复执行：

-   **暂停和恢复工作流**：使用[中断][langgraph.types.interrupt]函数在特定点暂停工作流，并使用[Command][langgraph.types.Command]原语通过更新的状态恢复它。有关更多详细信息，请参阅[**人工干预**](./human_in_the_loop.md)。
-   **从失败中恢复**：在异常（例如，LLM 提供商中断）后，通过提供 `None` 作为输入值来恢复具有相同线程标识符的工作流的执行（请参阅功能 API 的此[示例](../how-tos/use-functional-api.md#resuming-after-an-error)）。

## 恢复工作流的起点

*   如果您使用的是[StateGraph (Graph API)][langgraph.graph.state.StateGraph]，起点是执行停止的[**节点**](./low_level.md#nodes)的开头。
*   如果您在节点中调用子图，起点将是调用了挂起子图的**父**节点。
    在子图中，起点将是执行停止的特定[**节点**](./low_level.md#nodes)。
*   如果您使用的是 Functional API，起点将是执行停止的[**入口点**](./functional_api.md#entrypoint)的开头。