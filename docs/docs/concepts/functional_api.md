---
search:
  boost: 2
---

# 函数式 API 概念

## 概览

**函数式 API** 允许你通过最少的代码更改，将 LangGraph 的核心功能——[持久化](./persistence.md)、[内存](../how-tos/memory/add-memory.md)、[人工介入](./human_in_the_loop.md) 和 [流式输出](./streaming.md)——添加到你的应用程序中。

它旨在将这些功能集成到可能使用标准语言原生类型进行分支和控制流的现有代码中，例如 `if` 语句、`for` 循环和函数调用。与许多要求将代码重构为显式管道或 DAG 的数据编排框架不同，函数式 API 允许你在不强制执行严格执行模型的情况下整合这些功能。

函数式 API 使用两个关键构建块：

:::python

- **`@entrypoint`** – 将函数标记为工作流的起点，封装逻辑并管理执行流程，包括处理长时间运行的任务和中断。
- **`@task`** – 代表一个离散的工作单元，例如 API 调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似 future 的对象，可以同步等待或解析。
:::

:::js

- **`entrypoint`** – 入口点封装工作流逻辑并管理执行流程，包括处理长时间运行的任务和中断。
- **`task`** – 代表一个离散的工作单元，例如 API 调用或数据处理步骤，可以在入口点内异步执行。任务返回一个类似 future 的对象，可以同步等待或解析。
:::

这提供了一个最小化的抽象，用于构建具有状态管理和流式输出的工作流。

!!! tip

    有关如何使用函数式 API 的信息，请参阅[使用函数式 API](../how-tos/use-functional-api.md)。

## 函数式 API 与图 API

对于偏爱更声明式方法的用户，LangGraph 的[图 API](./low_level.md) 允许你使用图范式来定义工作流。这两种 API 共享相同的底层运行时，因此你可以在同一个应用程序中将它们结合使用。

以下是它们之间的一些关键区别：

- **控制流**：函数式 API 不需要考虑图结构。你可以使用标准的 Python 结构来定义工作流。这通常会减少你需要编写的代码量。
- **短期记忆**：**图 API** 要求声明一个 [**状态**](./low_level.md#state) 并且可能需要定义 [**reducers**](./low_level.md#reducers) 来管理图状态的更新。`@entrypoint` 和 `@tasks` 不需要显式状态管理，因为它们的状态作用域限制在函数内部，并且不会在函数之间共享。
- **检查点**：两种 API 都会生成并使用检查点。在**图 API** 中，每个 [超步](./low_level.md) 之后都会生成一个新的检查点。在**函数式 API** 中，当执行任务时，其结果会保存到与给定入口点关联的现有检查点中，而不是创建新检查点。
- **可视化**：图 API 可以轻松地将工作流可视化为图，这对于调试、理解工作流和与他人分享很有用。函数式 API 不支持可视化，因为图是在运行时动态生成的。

## 示例

下面我们演示一个简单的应用程序，该应用程序编写一篇论文，并通过[中断](./human_in_the_loop.md)请求人工审核。

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import interrupt

@task
def write_essay(topic: str) -> str:
    """Write an essay about the given topic."""
    time.sleep(1) # A placeholder for a long-running task.
    return f"An essay about topic: {topic}"

@entrypoint(checkpointer=InMemorySaver())
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

:::

:::js

```typescript
import { MemorySaver, entrypoint, task, interrupt } from "@langchain/langgraph";

const writeEssay = task("writeEssay", async (topic: string) => {
  // A placeholder for a long-running task.
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return `An essay about topic: ${topic}`;
});

const workflow = entrypoint(
  { checkpointer: new MemorySaver(), name: "workflow" },
  async (topic: string) => {
    const essay = await writeEssay(topic);
    const isApproved = interrupt({
      // Any json-serializable payload provided to interrupt as argument.
      // It will be surfaced on the client side as an Interrupt when streaming data
      // from the workflow.
      essay, // The essay we want reviewed.
      // We can add any additional information that we need.
      // For example, introduce a key called "action" with some instructions.
      action: "Please approve/reject the essay",
    });

    return {
      essay, // The essay that was generated
      isApproved, // Response from HIL
    };
  }
);
```

:::

??? example "详细解释"

    此工作流将围绕“猫”这一主题写一篇论文，然后暂停以获取人工审核。工作流可以无限期暂停，直到提供审核。

    当工作流恢复时，它将从头开始执行，但由于 `write_essay` 任务的结果已保存，因此任务结果将从检查点加载，而不会重新计算。

    :::python
    ```python
    import time
    import uuid
    from langgraph.func import entrypoint, task
    from langgraph.types import interrupt
    from langgraph.checkpoint.memory import InMemorySaver


    @task
    def write_essay(topic: str) -> str:
        """Write an essay about the given topic."""
        time.sleep(1)  # This is a placeholder for a long-running task.
        return f"An essay about topic: {topic}"

    @entrypoint(checkpointer=InMemorySaver())
    def workflow(topic: str) -> dict:
        """A simple workflow that writes an essay and asks for a review."""
        essay = write_essay("cat").result()
        is_approved = interrupt(
            {
                # Any json-serializable payload provided to interrupt as argument.
                # It will be surfaced on the client side as an Interrupt when streaming data
                # from the workflow.
                "essay": essay,  # The essay we want reviewed.
                # We can add any additional information that we need.
                # For example, introduce a key called "action" with some instructions.
                "action": "Please approve/reject the essay",
            }
        )
        return {
            "essay": essay,  # The essay that was generated
            "is_approved": is_approved,  # Response from HIL
        }


    thread_id = str(uuid.uuid4())
    config = {"configurable": {"thread_id": thread_id}}
    for item in workflow.stream("cat", config):
        print(item)
    # > {'write_essay': 'An essay about topic: cat'}
    # > {
    # >     '__interrupt__': (
    # >        Interrupt(
    # >            value={
    # >                'essay': 'An essay about topic: cat',
    # >                'action': 'Please approve/reject the essay'
    # >            },
    # >            id='b9b2b9d788f482663ced6dc755c9e981'
    # >        ),
    # >    )
    # > }
    ```

    一篇论文已经写好，准备好供审阅。一旦提供审稿，我们就可以恢复工作流：

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

    工作流已完成，评论已添加到论文中。
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { MemorySaver, entrypoint, task, interrupt } from "@langchain/langgraph";

    const writeEssay = task("writeEssay", async (topic: string) => {
      // This is a placeholder for a long-running task.
      await new Promise(resolve => setTimeout(resolve, 1000));
      return `An essay about topic: ${topic}`;
    });

    const workflow = entrypoint(
      { checkpointer: new MemorySaver(), name: "workflow" },
      async (topic: string) => {
        const essay = await writeEssay(topic);
        const isApproved = interrupt({
          // Any json-serializable payload provided to interrupt as argument.
          // It will be surfaced on the client side as an Interrupt when streaming data
          // from the workflow.
          essay, // The essay we want reviewed.
          // We can add any additional information that we need.
          // For example, introduce a key called "action" with some instructions.
          action: "Please approve/reject the essay",
        });

        return {
          essay, // The essay that was generated
          isApproved, // Response from HIL
        };
      }
    );

    const threadId = uuidv4();

    const config = {
      configurable: {
        thread_id: threadId
      }
    };

    for await (const item of workflow.stream("cat", config)) {
      console.log(item);
    }
    ```

    ```console
    { writeEssay: 'An essay about topic: cat' }
    {
      __interrupt__: [{
        value: { essay: 'An essay about topic: cat', action: 'Please approve/reject the essay' },
        resumable: true,
        ns: ['workflow:f7b8508b-21c0-8b4c-5958-4e8de74d2684'],
        when: 'during'
      }]
    }
    ```

    一篇论文已经写好，准备好供审阅。一旦提供审稿，我们就可以恢复工作流：

    ```typescript
    import { Command } from "@langchain/langgraph";

    // Get review from a user (e.g., via a UI)
    // In this case, we're using a bool, but this can be any json-sequentializable value.
    const humanReview = true;

    for await (const item of workflow.stream(new Command({ resume: humanReview }), config)) {
      console.log(item);
    }
    ```

    ```console
    { workflow: { essay: 'An essay about topic: cat', isApproved: true } }
    ```

    工作流已完成，评论已添加到论文中。
    :::

## 入口点

:::python
`@`[`@entrypoint`][entrypoint] 装饰器可用于从函数创建工作流。它封装工作流逻辑并管理执行流程，包括处理_长时间运行的任务_和[中断](./human_in_the_loop.md)。
:::

:::js
`entrypoint` 函数可用于从函数创建工作流。它封装工作流逻辑并管理执行流程，包括处理_长时间运行的任务_和[中断](./human_in_the_loop.md)。
:::

### 定义

:::python
**入口点**是通过使用 `@entrypoint` 装饰器装饰函数来定义的。

该函数**必须接受一个位置参数**，该参数充当工作流的输入。如果你需要传递多个数据，请使用字典作为第一个参数的输入类型。

使用 `entrypoint` 装饰函数会生成一个 @[`Pregel`][Pregel.stream] 实例，该实例有助于管理工作流的执行（例如，处理流式输出、恢复和检查点）。

你通常会希望将一个**检查点**传递给 `@entrypoint` 装饰器，以启用持久化并使用类似 **人工介入** 的功能。

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

:::

:::js
**入口点**是通过将配置和函数传递给 `entrypoint` 函数来定义的。

该函数**必须接受一个位置参数**，该参数充当工作流的输入。如果你需要传递多个数据，请使用对象作为第一个参数的输入类型。

使用函数创建入口点会生成工作流实例，该实例有助于管理工作流的执行（例如，处理流式输出、恢复和检查点）。

你通常会希望将一个**检查点**传递给 `entrypoint` 函数，以启用持久化并使用类似 **人工介入** 的功能。

```typescript
import { entrypoint } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (someInput: Record<string, any>): Promise<number> => {
    // some logic that may involve long-running tasks like API calls,
    // and may be interrupted for human-in-the-loop
    return result;
  }
);
```

:::

!!! important "序列化"

    入口点的**输入**和**输出**必须是 JSON 可序列化的，才能支持检查点。有关更多详细信息，请参阅[序列化](#serialization)部分。

:::python

### 可注入参数

在声明 `entrypoint` 时，你可以请求访问将在运行时自动注入的其他参数。这些参数包括：

| 参数    | 描述                                                                                                                                                        |
| ------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **previous** | 访问给定线程的先前 `checkpoint` 关联的状态。请参阅[短期记忆](#short-term-memory)。                                      |
| **store**    | [BaseStore][langgraph.store.base.BaseStore] 的实例。用于[长期记忆](../how-tos/use-functional-api.md#long-term-memory)。                      |
| **writer**   | 在使用 Async Python < 3.11 时用于访问 StreamWriter。有关详细信息，请参阅[使用函数式 API 进行流式输出](../how-tos/use-functional-api.md#streaming)。 |
| **config**   | 用于访问运行时配置。有关信息，请参阅[RunnableConfig](https://python.langchain.com/docs/concepts/runnables/#runnableconfig)。                  |

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

:::

### 执行

:::python
使用 [`@entrypoint`](#entrypoint) 会产生一个 @[`Pregel`][Pregel.stream] 对象，可以使用 `invoke`、`ainvoke`、`stream` 和 `astream` 方法执行。

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

:::

:::js
使用 [`entrypoint`](#entrypoint) 函数将返回一个对象，可以使用 `invoke` 和 `stream` 方法执行。

=== "Invoke"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };
    await myWorkflow.invoke(someInput, config); // Wait for the result
    ```

=== "Stream"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    for await (const chunk of myWorkflow.stream(someInput, config)) {
      console.log(chunk);
    }
    ```

:::

### 恢复

:::python
通过将 **resume** 值传递给 @[Command] 原语，可以在 @[interrupt][interrupt] 之后恢复执行。

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

:::

:::js
通过将 **resume** 值传递给 @[`Command`][Command] 原语，可以在 @[interrupt][interrupt] 之后恢复执行。

=== "Invoke"

    ```typescript
    import { Command } from "@langchain/langgraph";

    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    await myWorkflow.invoke(new Command({ resume: someResumeValue }), config);
    ```

=== "Stream"

    ```typescript
    import { Command } from "@langchain/langgraph";

    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    const stream = await myWorkflow.stream(
      new Command({ resume: someResumableValue }),
      config,
    )

    for await (const chunk of stream) {
      console.log(chunk);
    }
    ```

:::

:::python

**从错误中恢复**

要从错误中恢复，请使用 `None` 和相同的 **thread id**（配置）来运行 `entrypoint`。

这假设底层的 **错误** 已解决，并且执行可以成功进行。

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

:::

:::js

**从错误中恢复**

要从错误中恢复，请使用 `null` 和相同的 **thread id**（配置）来运行 `entrypoint`。

这假设底层的 **错误** 已解决，并且执行可以成功进行。

=== "Invoke"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    await myWorkflow.invoke(null, config);
    ```

=== "Stream"

    ```typescript
    const config = {
      configurable: {
        thread_id: "some_thread_id"
      }
    };

    for await (const chunk of myWorkflow.stream(null, config)) {
      console.log(chunk);
    }
    ```

:::

### 短期记忆

当 `entrypoint` 使用 `checkpointer` 定义时，它会在同一 **thread id** 的连续调用之间将信息存储在[检查点](persistence.md#checkpoints)中。

:::python
这使得可以使用 `previous` 参数访问先前调用的状态。

默认情况下，`previous` 参数是先前调用的返回值。

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

:::

:::js
这使得可以使用 `getPreviousState` 函数访问先前调用的状态。

默认情况下，`getPreviousState` 函数返回先前调用的返回值。

```typescript
import { entrypoint, getPreviousState } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (number: number) => {
    const previous = getPreviousState<number>() ?? 0;
    return number + previous;
  }
);

const config = {
  configurable: {
    thread_id: "some_thread_id",
  },
};

await myWorkflow.invoke(1, config); // 1 (previous was undefined)
await myWorkflow.invoke(2, config); // 3 (previous was 1 from the previous invocation)
```

:::

#### `entrypoint.final`

:::python
`@`[`entrypoint.final`][entrypoint.final] 是一个特殊的原语，可以从入口点返回，并允许**解耦****保存在检查点中的值**与**入口点的返回值**。

第一个值是入口点的返回值，第二个值是将被保存在检查点中的值。类型注解是 `entrypoint.final[return_type, save_type]`。

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

:::

:::js
`@`[`entrypoint.final`][entrypoint.final] 是一个特殊的原语，可以从入口点返回，并允许**解耦****保存在检查点中的值**与**入口点的返回值**。

第一个值是入口点的返回值，第二个值是将被保存在检查点中的值。

```typescript
import { entrypoint, getPreviousState } from "@langchain/langgraph";

const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (number: number) => {
    const previous = getPreviousState<number>() ?? 0;
    // This will return the previous value to the caller, saving
    // 2 * number to the checkpoint, which will be used in the next invocation
    // for the `previous` parameter.
    return entrypoint.final({
      value: previous,
      save: 2 * number,
    });
  }
);

const config = {
  configurable: {
    thread_id: "1",
  },
};

await myWorkflow.invoke(3, config); // 0 (previous was undefined)
await myWorkflow.invoke(1, config); // 6 (previous was 3 * 2 from the previous invocation)
```

:::

## 任务

**任务**代表一个离散的工作单元，例如 API 调用或数据处理步骤。它有两个关键特征：

- **异步执行**：任务旨在异步执行，允许多个操作并发运行而不阻塞。
- **检查点**：任务结果会被保存到检查点，以便能够在从最后一个保存状态恢复工作流。 (有关更多详细信息，请参阅[持久化](persistence.md))。

### 定义

:::python
任务使用 `@task` 装饰器定义，该装饰器包装了一个常规的 Python 函数。

```python
from langgraph.func import task

@task()
def slow_computation(input_value):
    # Simulate a long-running operation
    ...
    return result
```

:::

:::js
任务使用 `task` 函数定义，该函数包装了一个常规函数。

```typescript
import { task } from "@langchain/langgraph";

const slowComputation = task("slowComputation", async (inputValue: any) => {
  // Simulate a long-running operation
  return result;
});
```

:::

!!! important "序列化"

    任务的**输出**必须是 JSON 可序列化的，才能支持检查点。

### 执行

**任务**只能在**入口点**、另一个**任务**或[状态图节点](./low_level.md#nodes)内调用。

任务**不能**直接从主应用程序代码调用。

:::python
当你调用一个**任务**时，它会_立即_返回一个 future 对象。Future 是一个占位符，用于稍后可用的结果。

要获取**任务**的结果，你可以使用 `result()` 同步等待它，或者使用 `await` 异步等待它。

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

:::

:::js
当你调用一个**任务**时，它返回一个可以被 await 的 Promise。

```typescript
const myWorkflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (someInput: number): Promise<number> => {
    return await slowComputation(someInput);
  }
);
```

:::

## 何时使用任务

**任务**在以下场景中很有用：

- **检查点**：当你需要将长时间运行的操作的结果保存到检查点时，这样在恢复工作流时就不需要重新计算。
- **人工介入**：如果你正在构建需要人工干预的工作流，则**必须**使用**任务**来封装任何随机性（例如 API 调用），以确保工作流能够正确恢复。有关更多详细信息，请参阅[确定性](#determinism)部分。
- **并行执行**：对于 I/O 密集型任务，**任务**支持并行执行，允许多个操作并发运行而不阻塞（例如，调用多个 API）。
- **可观测性**：将操作封装在**任务**中提供了一种方式，可以通过[LangSmith](https://docs.smith.langchain.com/) 来跟踪工作流的进度和监控单个操作的执行。
- **可重试任务**：当工作需要重试以处理失败或不一致时，**任务**提供了一种封装和管理重试逻辑的方式。

## 序列化

LangGraph 中的序列化有两个关键方面：

1. `entrypoint` 的输入和输出必须是 JSON 可序列化的。
2. `task` 的输出必须是 JSON 可序列化的。

:::python
这些要求对于启用检查点和工作流恢复至关重要。请使用字典、列表、字符串、数字和布尔值等 Python 原语来确保你的输入和输出是可序列化的。
:::

:::js
这些要求对于启用检查点和工作流恢复至关重要。请使用对象、数组、字符串、数字和布尔值等原始类型来确保你的输入和输出是可序列化的。
:::

序列化确保工作流状态，例如任务结果和中间值，可以被可靠地保存和恢复。这对于实现人工介入、容错和并行执行至关重要。

如果提供不可序列化的输入或输出，当使用检查点配置工作流时，将导致运行时错误。

## 确定性

为了利用**人工介入**等功能，任何随机性都应该封装在**任务**内部。这可以确保当执行被暂停（例如，为了人工介入）然后恢复时，它将遵循相同的_步骤序列_，即使**任务**结果不是确定性的。

LangGraph 通过在执行时持久化**任务**和[**子图**](./subgraphs.md)的结果来实现这一点。精心设计的工作流可以确保恢复执行遵循_相同的步骤序列_，允许在不重新执行的情况下正确检索先前计算的结果。这对于长时间运行的**任务**或具有非确定性结果的**任务**特别有用，因为它可以避免重复先前完成的工作并允许从与原始执行基本相同的点恢复。

虽然工作流的不同运行可能会产生不同的结果，但**特定**运行的恢复应始终遵循相同的记录步骤序列。这使得 LangGraph 能够有效地查找在图被中断之前执行的**任务**和**子图**结果，并避免重新计算它们。

## 幂等性

幂等性确保多次运行同一操作会产生相同的结果。这有助于防止重复调用 API 和冗余处理，以防步骤因失败而重新执行。始终将 API 调用放在 **task** 函数内部以进行检查点，并设计它们是幂等的，以防重新执行。如果一个**task**开始但未成功完成，则可能发生重新执行。然后，如果恢复工作流，该**task**将再次运行。使用幂等性键或验证现有结果以避免重复。

## 常见陷阱

### 处理副作用

将副作用（例如，写入文件、发送电子邮件）封装在任务中，以确保它们在恢复工作流时不会被执行多次。

=== "错误"

    在此示例中，副作用（写入文件）直接包含在工作流中，因此在恢复工作流时它将被执行第二次。

    :::python
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
    :::

    :::js
    ```typescript
    import { entrypoint, interrupt } from "@langchain/langgraph";
    import fs from "fs";

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: Record<string, any>) => {
        // This code will be executed a second time when resuming the workflow.
        // Which is likely not what you want.
        fs.writeFileSync("output.txt", "Side effect executed");
        const value = interrupt("question");
        return value;
      }
    );
    ```
    :::

=== "正确"

    在此示例中，副作用被封装在一个任务中，确保恢复时的一致执行。

    :::python
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
    :::

    :::js
    ```typescript
    import { entrypoint, task, interrupt } from "@langchain/langgraph";
    import * as fs from "fs";

    const writeToFile = task("writeToFile", async () => {
      fs.writeFileSync("output.txt", "Side effect executed");
    });

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: Record<string, any>) => {
        // The side effect is now encapsulated in a task.
        await writeToFile();
        const value = interrupt("question");
        return value;
      }
    );
    ```
    :::

### 非确定性控制流

每次可能产生不同结果的操作（如获取当前时间或随机数）应封装在任务中，以确保恢复时返回相同的结果。

- 在任务中：获取随机数 (5) → 中断 → 恢复 → (再次返回 5) → ...
- 不在任务中：获取随机数 (5) → 中断 → 恢复 → 获取新随机数 (7) → ...

:::python
这在使用具有多个中断调用的人工介入工作流时尤为重要。LangGraph 维护每个任务/入口点的恢复值列表。遇到中断时，它会与相应的恢复值匹配。此匹配是严格按_索引_进行的，因此恢复值的顺序应与中断的顺序匹配。
:::

:::js
这在使用具有多个中断调用的人工介入工作流时尤为重要。LangGraph 维护每个任务/入口点的恢复值列表。遇到中断时，它会与相应的恢复值匹配。此匹配是严格按_索引_进行的，因此恢复值的顺序应与中断的顺序匹配。
:::

如果在恢复时未保持执行顺序，一个 `interrupt` 调用可能会与错误的 `resume` 值匹配，从而导致不正确的结果。

有关更多详细信息，请阅读[确定性](#determinism)部分。

=== "错误"

    在此示例中，工作流使用当前时间来确定要执行哪个任务。这是非确定性的，因为工作流的结果取决于其执行时间。

    :::python
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
    :::

    :::js
    ```typescript
    import { entrypoint, interrupt } from "@langchain/langgraph";

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: { t0: number }) => {
        const t1 = Date.now();

        const deltaT = t1 - inputs.t0;

        if (deltaT > 1000) {
          const result = await slowTask(1);
          const value = interrupt("question");
          return { result, value };
        } else {
          const result = await slowTask(2);
          const value = interrupt("question");
          return { result, value };
        }
      }
    );
    ```
    :::

=== "正确"

    :::python
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
    :::

    :::js
    在此示例中，工作流使用输入 `t0` 来确定要执行哪个任务。这是确定性的，因为工作流的结果仅取决于输入。

    ```typescript
    import { entrypoint, task, interrupt } from "@langchain/langgraph";

    const getTime = task("getTime", () => Date.now());

    const myWorkflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: { t0: number }): Promise<any> => {
        const t1 = await getTime();

        const deltaT = t1 - inputs.t0;

        if (deltaT > 1000) {
          const result = await slowTask(1);
          const value = interrupt("question");
          return { result, value };
        } else {
          const result = await slowTask(2);
          const value = interrupt("question");
          return { result, value };
        }
      }
    );
    ```
    :::