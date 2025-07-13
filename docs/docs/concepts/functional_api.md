---
search:
  boost: 2
---

# Functional API 概念

## 概述

**Functional API** 允许您通过最少的代码更改，将 LangGraph 的核心功能 — [持久化](./persistence.md)、[内存](../how-tos/memory/add-memory.md)、[人工干预](./human_in_the_loop.md) 和 [流式传输](./streaming.md) — 添加到您的应用程序中。

它旨在将这些功能集成到可能使用标准语言原语进行分支和控制流的现有代码中，例如 `if` 语句，`for` 循环和函数调用。与许多强制要求将代码重构为显式管道或 DAG 的数据编排框架不同，Functional API 允许您在不强制执行严格执行模型的情况下结合这些功能。

Functional API 使用两个关键构建块：

- **`@entrypoint`** – 将函数标记为工作流的起点，封装逻辑并管理执行流，包括处理长期运行的任务和中断。
- **`@task`** – 表示离散的工作单元，例如 API 调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似 future 的对象，可以同步等待或解析。

这为构建具有状态管理和流式传输的工作流提供了一个最小化的抽象。

!!! tip

    有关如何使用 Functional API 的信息，请参阅 [使用 Functional API](../how-tos/use-functional-api.md)。

## Functional API 与 Graph API 对比

对于偏爱更声明式方法的用户，LangGraph 的 [Graph API](./low_level.md) 允许您使用 Graph 范例定义工作流。两个 API 使用相同的底层运行时，因此您可以在同一个应用程序中一起使用它们。

以下是一些主要区别：

- **控制流**：Functional API 不需要考虑图结构。您可以使用标准的 Python 构造来定义工作流。这通常会减少您需要编写的代码量。
- **短期内存**：**Graph API** 需要声明一个 [**状态**](./low_level.md#state)，并且可能需要定义 [** reducers**](./low_level.md#reducers) 来管理图状态的更新。`@entrypoint` 和 `@tasks` 不需要显式状态管理，因为它们的状态范围限定在函数内，并且不跨函数共享。
"- **检查点**：两个 API 都会生成和使用检查点。在 **Graph API** 中，每个 [superstep](./low_level.md) 之后都会生成一个新的检查点。在 **Functional API** 中，当执行任务时，其结果会被保存​​到与给定入口点关联的现有检查点中，而不是创建新检查点。
- **可视化**：Graph API 可以轻松地将工作流可视化为图，这对于调试、理解工作流和与他人共享很有用。Functional API 不支持可视化，因为图是在运行时动态生成的。

## 示例

下面我们演示一个简单的应用程序，该应用程序撰写一篇论文并[中断](./human_in_the_loop.md)以请求人工审查。

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt


@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    time.sleep(1) # A placeholder for a long-running task.
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=MemorySaver())
def workflow(topic: str) -> dict:
    """A simple workflow that writes an essay and asks for a review."""
    essay = write_essay("cat").result()
    is_approved = interrupt({
        # Any json-serializable payload provided to interrupt as argument.
        # It will be surfaced on the client side as an Interrupt when streaming data
        # from the workflow.
        "essay": essay, # The essay we want reviewed.
        # We can add any additional information that we need.
        # For example, introduce a key called "action" with some instructions.
        "action": "Please approve/reject the essay",
    })

    return {
        "essay": essay, # The essay that was generated
        "is_approved": is_approved, # Response from HIL
    }
```

??? example "详细说明"

    此工作流将围绕主题“cat”写一篇论文，然后暂停以获取人工审查。工作流可以无限期地中断，直到提供审查。

    恢复工作流时，它会从头开始执行，但由于 `write_essay` 任务的结果已被保存，因此将从检查点加载任务结果，而不是重新计算。

    ```python
    import time
    import uuid

    from langgraph.func import entrypoint, task
    from langgraph.types import interrupt
    from langgraph.checkpoint.memory import MemorySaver

    @task
    def write_essay(topic: str) -> str:
        """Write an essay about the given topic."""
        time.sleep(1) # This is a placeholder for a long-running task.
        return f"An essay about topic: {topic}"

    @entrypoint(checkpointer=MemorySaver())
    def workflow(topic: str) -> dict:
        """A simple workflow that writes an essay and asks for a review."""
        essay = write_essay("cat").result()
        is_approved = interrupt({
            # Any json-serializable payload provided to interrupt as argument.
            # It will be surfaced on the client side as an Interrupt when streaming data
            # from the workflow.
            "essay": essay, # The essay we want reviewed.
            # We can add any additional information that we need.
            # For example, introduce a key called "action" with some instructions.
            "action": "Please approve/reject the essay",
        })

        return {
            "essay": essay, # The essay that was generated
            "is_approved": is_approved, # Response from HIL
        }

    thread_id = str(uuid.uuid4())

    config = {
        "configurable": {
            "thread_id": thread_id
        }
    }

    for item in workflow.stream("cat", config):
        print(item)
    ```

    ```pycon
    {'write_essay': 'An essay about topic: cat'}
    {'__interrupt__': (Interrupt(value={'essay': 'An essay about topic: cat', 'action': 'Please approve/reject the essay'}, resumable=True, ns=['workflow:f7b8508b-21c0-8b4c-5958-4e8de74d2684'], when='during'),)}
    ```

    文章已写好并准备审查。一旦提供审查，我们就可以恢复工作流：

    ```python
    from langgraph.types import Command

    # Get review from a user (e.g., via a UI)
    # In this case, we're using a bool, but this can be any json-serializable value.
    human_review = True

    for item in workflow.stream(Command(resume=human_review), config):
        print(item)
    ```

    ```pycon
    {'workflow': {'essay': 'An essay about topic: cat', 'is_approved': False}}
    ```

    工作流已完成，审查已添加到论文中。

## Entrypoint

[`@entrypoint`][langgraph.func.entrypoint] 装饰器可用于从函数创建工作流。它封装了工作流逻辑并管理执行流，包括处理*长期运行的任务*和[中断](./human_in_the_loop.md)。

### 定义

**入口点**通过使用 `@entrypoint` 装饰器来定义一个函数。

该函数**必须接受一个位置参数**，该参数用作工作流输入。如果您需要传递多个数据，请使用字典作为第一个参数的输入类型。

使用 `entrypoint` 装饰函数会生成一个 [`Pregel`][langgraph.pregel.Pregel.stream] 实例，该实例有助于管理工作流的执行（例如，处理流式传输、恢复和检查点）。

您通常希望将**检查点**传递给 `@entrypoint` 装饰器，以启用持久化并使用诸如**人工干预**之类的功能。

=== "同步"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop.
        ...
        return result
    ```

=== "异步"

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: dict) -> int:
        # some logic that may involve long-running tasks like API calls,
        # and may be interrupted for human-in-the-loop
        ...
        return result
    ```

!!! important "序列化"

    入口点的**输入**和**输出**必须是 JSON 序列化的，以支持检查点。请参阅[序列化](#serialization)部分了解更多详细信息。

### 可注入参数

在声明 `entrypoint` 时，您可以请求访问将在运行时自动注入的其他参数。这些参数包括：

| 参数     | 描述                                                                                                                                                            |
|--------------|--------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| **previous** | 访问给定线程上一次 `checkpoint` 关联的状态。请参阅[短期内存](#short-term-memory)。                                                                                                                                                              |
| **store**    | [BaseStore][langgraph.store.base.BaseStore] 的实例。对[长期内存](../how-tos/use-functional-api.md#long-term-memory)很有用。                                                                                                                                                                                                                                       |
| **writer**   | 使用此参数访问 StreamWriter，以便在处理 Async Python < 3.11 时能够使用。有关详细信息，请参阅[使用 Functional API 进行流式传输](../how-tos/use-functional-api.md#streaming)。                                                                                                                                                                                                                                                |
| **config**   | 用于访问运行时的配置。有关信息，请参阅[RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig)。                                                                                                                                                                                                                                                  |

!!! important

    使用适当的名称和类型注解声明参数。

??? example "请求可注入参数"

    ```python
    from langchain_core.runnables import RunnableConfig
    from langgraph.func import entrypoint
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore

    in_memory_store = InMemoryStore(...)  # An instance of InMemoryStore for long-term memory

    @entrypoint(
        checkpointer=checkpointer,  # Specify the checkpointer
        store=in_memory_store  # Specify the store
    )
    def my_workflow(
        some_input: dict,  # The input (e.g., passed via `invoke`)
        *,
        previous: Any = None, # For short-term memory
        store: BaseStore,  # For long-term memory
        writer: StreamWriter,  # For streaming custom data
        config: RunnableConfig  # For accessing the configuration passed to the entrypoint
    ) -> ...:
    ```

### 执行

使用 [`@entrypoint`](#entrypoint) 会产生一个 [`Pregel`][langgraph.pregel.Pregel.stream] 对象，可以使用 `invoke`、`ainvoke`、`stream` 和 `astream` 方法对其进行执行。

=== "Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    my_workflow.invoke(some_input, config)  # Wait for the result synchronously
    ```

=== "Async Invoke"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }
    await my_workflow.ainvoke(some_input, config)  # Await result asynchronously
    ```

=== "Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(some_input, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(some_input, config):
        print(chunk)
    ```

### 恢复

可以通过将**恢复**值传递给 [Command][langgraph.types.Command] 原始类型来恢复[中断][langgraph.types.interrupt]后的执行。

=== "Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(Command(resume=some_resume_value), config)
    ```

=== "Async Invoke"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(Command(resume=some_resume_value), config)
    ```

=== "Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

=== "Async Stream"

    ```python
    from langgraph.types import Command

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(Command(resume=some_resume_value), config):
        print(chunk)
    ```

**恢复错误后**

要从错误中恢复，请使用 `None` 和相同的**线程 ID**（配置）运行 `entrypoint`。

这假设底层的**错误**已解决，并且执行可以成功继续。

=== "Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    my_workflow.invoke(None, config)
    ```

=== "Async Invoke"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    await my_workflow.ainvoke(None, config)
    ```

=== "Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    for chunk in my_workflow.stream(None, config):
        print(chunk)
    ```

=== "Async Stream"

    ```python

    config = {
        "configurable": {
            "thread_id": "some_thread_id"
        }
    }

    async for chunk in my_workflow.astream(None, config):
        print(chunk)
    ```

### 短期记忆

当使用 `checkpointer` 定义 `entrypoint` 时，它会在同一**线程 ID** 上的连续调用之间将信息存储在[检查点](persistence.md#checkpoints)中。

这使得可以使用 `previous` 参数从前一个调用中访问状态。

默认情况下，`previous` 参数是前一个调用的返回值。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> int:
    previous = previous or 0
    return number + previous

config = {
    "configurable": {
        "thread_id": "some_thread_id"
    }
}

my_workflow.invoke(1, config)  # 1 (previous was None)
my_workflow.invoke(2, config)  # 3 (previous was 1 from the previous invocation)
```

#### `entrypoint.final`

[entrypoint.final][langgraph.func.entrypoint.final] 是可以从入口点返回的特殊原语，它允许**解耦**要在**检查点中保存**的值与**入口点的返回值**。

第一个值是入口点的返回值，第二个值是要保存在检查点中的值。类型注解为 `entrypoint.final[return_type, save_type]`。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(number: int, *, previous: Any = None) -> entrypoint.final[int, int]:
    previous = previous or 0
    # This will return the previous value to the caller, saving
    # 2 * number to the checkpoint, which will be used in the next invocation
    # for the `previous` parameter.
    return entrypoint.final(value=previous, save=2 * number)

config = {
    "configurable": {
        "thread_id": "1"
    }
}

my_workflow.invoke(3, config)  # 0 (previous was None)
my_workflow.invoke(1, config)  # 6 (previous was 3 * 2 from the previous invocation)
```

## Task

**任务**代表离散的工作单元，例如 API 调用或数据处理步骤。它具有两个关键特征：

* **异步执行**：任务设计用于异步执行，允许多个操作并发运行而不阻塞。
* **检查点**：任务结果被保存到检查点，从而能够从最后一个保存的状态恢复工作流。（有关更多详细信息，请参阅[持久化](persistence.md)）。

### 定义

任务使用 `@task` 装饰器定义，该装饰器包装了一个常规的 Python 函数。

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    # Simulate a long-running operation
    ...
    return result
```

!!! important "序列化"

    任务的**输出**必须是 JSON 序列化的，以支持检查点。

### 执行

**任务**只能在**入口点**、另一个**任务**或[状态图节点](./low_level.md#nodes)内部调用。

**任务**不能直接从主应用程序代码调用。

当您调用一个**任务**时，它会立即返回一个 future 对象。Future 是稍后将可用结果的占位符。

要获取**任务**的结果，您可以同步等待（使用 `result()`）或异步等待（使用 `await`）。

=== "同步调用"

    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(some_input: int) -> int:
        future = slow_computation(some_input)
        return future.result()  # Wait for the result synchronously
    ```

=== "异步调用"

    ```python
    @entrypoint(checkpointer=checkpointer)
    async def my_workflow(some_input: int) -> int:
        return await slow_computation(some_input)  # Await result asynchronously
    ```

## 何时使用任务

**任务**在以下场景中很有用：

- **检查点**：当您需要将长期运行操作的结果保存到检查点时，以便在恢复工作流时不必重新计算它。
- **人工干预**：如果您正在构建需要人工干预的工作流，则必须使用**任务**来封装任何随机性（例如，API 调用），以确保工作流能够正确恢复。有关更多详细信息，请参阅[确定性](#determinism)部分。
- **并行执行**：对于 I/O 密集型任务，**任务**支持并行执行，允许多个操作并发运行而不阻塞（例如，调用多个 API）。
- **可观测性**：将操作包装在**任务**中，可以通过[LangSmith](https://docs.smith.langchain.com/) 提供一种跟踪工作流进度和监控单个操作执行的方法。
- **可重试工作**：当需要重试工作以处理失败或不一致时，**任务**提供了一种封装和管理重试逻辑的方法。

## 序列化

LangGraph 中的序列化有两个关键方面：

1. `@entrypoint` 输入和输出必须是 JSON 序列化的。
2. `@task` 输出必须是 JSON 序列化的。

这些要求对于启用检查点和工作流恢复至关重要。使用 Python 原始类型，如字典、列表、字符串、数字和布尔值，以确保您的输入和输出是可序列化的。

序列化可确保工作流状态（如任务结果和中间值）能够被可靠地保存和恢复。这对于启用人工干预交互、容错和并行执行至关重要。

提供不可序列化的输入或输出，在配置了检查点的​​工作流运行时会导致错误。

## 确定性

为了利用**人工干预**等功能，任何随机性都应封装在**任务**内部。这可以确保当执行暂停（例如，为了人工干预）然后恢复时，即使**任务**结果不是确定性的，它也会遵循相同的*步骤序列*。

LangGraph 通过在执行时持久化**任务**和[**子图**](./subgraphs.md)结果来实现此行为。设计良好的工作流可确保恢复执行遵循*相同的步骤序列*，从而允许在不重新执行的情况下正确检索先前计算的结果。这对于长期运行的**任务**或具有非确定性结果的**任务**特别有用，因为它避免了重复先前完成的工作，并允许从几乎相同的位置恢复。

虽然工作流的不同运行可能会产生不同的结果，但恢复*特定*运行应始终遵循相同的已记录步骤序列。这使得 LangGraph 能够有效地查找图被中断之前执行的**任务**和**子图**结果，并避免重新计算它们。

## 幂等性

幂等性确保多次运行相同的操作会产生相同的结果。这有助于防止重复的 API 调用和冗余处理，如果某个步骤因失败而重新执行。始终将 API 调用放在**任务**函数中以进行检查点，并根据需要设计它们以在重新执行时具有幂等性。重新执行可能发生在任务开始但未成功完成时。然后，如果恢复工作流，任务将再次运行。使用幂等性键或验证现有结果以避免重复。

## 常见陷阱

### 处理副作用

将副作用（例如，写入文件、发送电子邮件）封装在任务中，以确保在恢复工作流时不会多次执行它们。

=== "不正确"

    在此示例中，副作用（写入文件）直接包含在工作流中，因此在恢复工作流时将执行第二次。

    ```python
    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # This code will be executed a second time when resuming the workflow.
        # Which is likely not what you want.
        # highlight-next-line
        with open("output.txt", "w") as f:
            # highlight-next-line
            f.write("Side effect executed")
        value = interrupt("question")
        return value
    ```

=== "正确"

    在此示例中，副作用已封装在任务中，从而确保在恢复时执行一致。

    ```python
    from langgraph.func import task

    # highlight-next-line
    @task
    # highlight-next-line
    def write_to_file():
        with open("output.txt", "w") as f:
            f.write("Side effect executed")

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        # The side effect is now encapsulated in a task.
        write_to_file().result()
        value = interrupt("question")
        return value
    ```

### 非确定性控制流

每次可能产生不同结果的操作（例如，获取当前时间或随机数）应封装在任务中，以确保恢复时返回相同的结果。

* 在任务中：获取随机数 (5) → 中断 → 恢复 → (再次返回 5) → ...
* 不在任务中：获取随机数 (5) → 中断 → 恢复 → 获取新的随机数 (7) → ...

当在具有多个中断调用的**人工干预**工作流中使用时，这一点尤其重要。LangGraph 会维护一个每个任务/入口点的恢复值列表。遇到中断时，它会与相应的恢复值匹配。此匹配严格基于*索引*，因此恢复值的顺序应与中断的顺序匹配。

如果恢复时未维护执行顺序，一个 `interrupt` 调用可能会与错误的 `resume` 值匹配，从而导致结果不正确。

有关更多详细信息，请阅读[确定性](#determinism)部分。

=== "不正确"

    在此示例中，工作流使用当前时间来确定要执行哪个任务。这是非确定性的，因为工作流的结果取决于其执行的时间。

    ```python
    from langgraph.func import entrypoint

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        # highlight-next-line
        t1 = time.time()

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```

=== "正确"

    在此示例中，工作流使用输入 `t0` 来确定要执行哪个任务。这是确定性的，因为工作流的结果仅取决于输入。

    ```python
    import time

    from langgraph.func import task

    # highlight-next-line
    @task
    # highlight-next-line
    def get_time() -> float:
        return time.time()

    @entrypoint(checkpointer=checkpointer)
    def my_workflow(inputs: dict) -> int:
        t0 = inputs["t0"]
        # highlight-next-line
        t1 = get_time().result()

        delta_t = t1 - t0

        if delta_t > 1:
            result = slow_task(1).result()
            value = interrupt("question")
        else:
            result = slow_task(2).result()
            value = interrupt("question")

        return {
            "result": result,
            "value": value
        }
    ```