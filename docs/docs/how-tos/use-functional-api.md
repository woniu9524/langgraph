# 使用函数式 API (Functional API)

[**函数式 API**](../concepts/functional_api.md) 允许你通过少量代码改动，将 LangGraph 的核心功能——[持久化](../concepts/persistence.md)、[记忆](../how-tos/memory/add-memory.md)、[人工干预](../concepts/human_in_the_loop.md) 和 [流式输出](../concepts/streaming.md)—集成到现有应用中。

!!! tip

    关于函数式 API 的概念信息，请参阅 [函数式 API](../concepts/functional_api.md)。

## 创建一个简单的流程

在定义 `entrypoint` 时，输入仅限于函数的第一个参数。要传递多个输入，可以使用字典。

:::python

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = inputs["value"]
    another_value = inputs["another_value"]
    ...

my_workflow.invoke({"value": 1, "another_value": 2})
```

:::

:::js

```typescript
const checkpointer = new MemorySaver();

const myWorkflow = entrypoint(
  { checkpointer, name: "myWorkflow" },
  async (inputs: { value: number; anotherValue: number }) => {
    const value = inputs.value;
    const anotherValue = inputs.anotherValue;
    // ...
  }
);

await myWorkflow.invoke({ value: 1, anotherValue: 2 });
```

:::

??? example "扩展示例：简单流程"

    :::python
    ```python
    import uuid
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import InMemorySaver

    # 检查数字是否为偶数的任务
    @task
    def is_even(number: int) -> bool:
        return number % 2 == 0

    # 格式化消息的任务
    @task
    def format_message(is_even: bool) -> str:
        return "The number is even." if is_even else "The number is odd."

    # 创建一个用于持久化的 checkpointer
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(inputs: dict) -> str:
        """用于分类数字的简单流程。"""
        even = is_even(inputs["number"]).result()
        return format_message(even).result()

    # 使用唯一的 thread ID 运行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    result = workflow.invoke({"number": 7}, config=config)
    print(result)
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { entrypoint, task, MemorySaver } from "@langchain/langgraph";

    // 检查数字是否为偶数的任务
    const isEven = task("isEven", async (number: number) => {
      return number % 2 === 0;
    });

    // 格式化消息的任务
    const formatMessage = task("formatMessage", async (isEven: boolean) => {
      return isEven ? "The number is even." : "The number is odd.";
    });

    // 创建一个用于持久化的 checkpointer
    const checkpointer = new MemorySaver();

    const workflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (inputs: { number: number }) => {
        // 用于分类数字的简单流程
        const even = await isEven(inputs.number);
        return await formatMessage(even);
      }
    );

    // 使用唯一的 thread ID 运行流程
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await workflow.invoke({ number: 7 }, config);
    console.log(result);
    ```
    :::

??? example "扩展示例：用 LLM 创作文章"

    此示例在句法上演示了如何使用 `@task` 和 `@entrypoint` 装饰器。如果提供了 checkpointer，则流程结果将被持久化到 checkpointer 中。

    :::python
    ```python
    import uuid
    from langchain.chat_models import init_chat_model
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import InMemorySaver

    llm = init_chat_model('openai:gpt-3.5-turbo')

    # 任务：使用 LLM 生成文章
    @task
    def compose_essay(topic: str) -> str:
        """关于给定主题生成文章。"""
        return llm.invoke([
            {"role": "system", "content": "You are a helpful assistant that writes essays."},
            {"role": "user", "content": f"Write an essay about {topic}."}
        ]).content

    # 创建一个用于持久化的 checkpointer
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(topic: str) -> str:
        """使用 LLM 生成文章的简单流程。"""
        return compose_essay(topic).result()

    # 执行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    result = workflow.invoke("the history of flight", config=config)
    print(result)
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { ChatOpenAI } from "@langchain/openai";
    import { entrypoint, task, MemorySaver } from "@langchain/langgraph";

    const llm = new ChatOpenAI({ model: "gpt-3.5-turbo" });

    // 任务：使用 LLM 生成文章
    const composeEssay = task("composeEssay", async (topic: string) => {
      // 关于给定主题生成文章
      const response = await llm.invoke([
        { role: "system", content: "You are a helpful assistant that writes essays." },
        { role: "user", content: `Write an essay about ${topic}.` }
      ]);
      return response.content as string;
    });

    // 创建一个用于持久化的 checkpointer
    const checkpointer = new MemorySaver();

    const workflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (topic: string) => {
        // 使用 LLM 生成文章的简单流程
        return await composeEssay(topic);
      }
    );

    // 执行流程
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await workflow.invoke("the history of flight", config);
    console.log(result);
    ```
    :::

## 并行执行

可以通过并发调用任务并等待结果来实现任务的并行执行。这对于提高 IO 密集型任务（例如调用 LLM API）的性能非常有用。

:::python

```python
@task
def add_one(number: int) -> int:
    return number + 1

@entrypoint(checkpointer=checkpointer)
def graph(numbers: list[int]) -> list[str]:
    futures = [add_one(i) for i in numbers]
    return [f.result() for f in futures]
```

:::

:::js

```typescript
const addOne = task("addOne", async (number: number) => {
  return number + 1;
});

const graph = entrypoint(
  { checkpointer, name: "graph" },
  async (numbers: number[]) => {
    return await Promise.all(numbers.map(addOne));
  }
);
```

:::

??? example "扩展示例：并行 LLM 调用"

    此示例演示了如何使用 `@task` 并行运行多个 LLM 调用。每次调用都会生成不同主题的段落，并将结果合并为单个文本输出。

    :::python
    ```python
    import uuid
    from langchain.chat_models import init_chat_model
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import InMemorySaver

    # 初始化 LLM 模型
    llm = init_chat_model("openai:gpt-3.5-turbo")

    # 关于给定主题生成段落的任务
    @task
    def generate_paragraph(topic: str) -> str:
        response = llm.invoke([
            {"role": "system", "content": "You are a helpful assistant that writes educational paragraphs."},
            {"role": "user", "content": f"Write a paragraph about {topic}."}
        ])
        return response.content

    # 创建一个用于持久化的 checkpointer
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(topics: list[str]) -> str:
        """并行生成多个段落并进行合并。"""
        futures = [generate_paragraph(topic) for topic in topics]
        paragraphs = [f.result() for f in futures]
        return "\n\n".join(paragraphs)

    # 运行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    result = workflow.invoke(["quantum computing", "climate change", "history of aviation"], config=config)
    print(result)
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { ChatOpenAI } from "@langchain/openai";
    import { entrypoint, task, MemorySaver } from "@langchain/langgraph";

    // 初始化 LLM 模型
    const llm = new ChatOpenAI({ model: "gpt-3.5-turbo" });

    // 关于给定主题生成段落的任务
    const generateParagraph = task("generateParagraph", async (topic: string) => {
      const response = await llm.invoke([
        { role: "system", content: "You are a helpful assistant that writes educational paragraphs." },
        { role: "user", content: `Write a paragraph about ${topic}.` }
      ]);
      return response.content as string;
    });

    // 创建一个用于持久化的 checkpointer
    const checkpointer = new MemorySaver();

    const workflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (topics: string[]) => {
        // 并行生成多个段落并进行合并
        const paragraphs = await Promise.all(topics.map(generateParagraph));
        return paragraphs.join("\n\n");
      }
    );

    // 运行流程
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await workflow.invoke(["quantum computing", "climate change", "history of aviation"], config);
    console.log(result);
    ```
    :::

    此示例利用 LangGraph 的并发模型来提高执行时间，尤其是在任务涉及 LLM 完成等 I/O 操作时。

## 调用图 (Graphs)

**函数式 API** 和 [**图 API**](../concepts/low_level.md) 可以在同一个应用程序中协同使用，因为它们共享相同的底层运行时。

:::python

```python
from langgraph.func import entrypoint
from langgraph.graph import StateGraph

builder = StateGraph()
...
some_graph = builder.compile()

@entrypoint()
def some_workflow(some_input: dict) -> int:
    # 调用使用图 API 定义的图
    result_1 = some_graph.invoke(...)
    # 调用另一个使用图 API 定义的图
    result_2 = another_graph.invoke(...)
    return {
        "result_1": result_1,
        "result_2": result_2
    }
```

:::

:::js

```typescript
import { entrypoint } from "@langchain/langgraph";
import { StateGraph } from "@langchain/langgraph";

const builder = new StateGraph(/* ... */);
// ...
const someGraph = builder.compile();

const someWorkflow = entrypoint(
  { name: "someWorkflow" },
  async (someInput: Record<string, any>) => {
    // 调用使用图 API 定义的图
    const result1 = await someGraph.invoke(/* ... */);
    // 调用另一个使用图 API 定义的图
    const result2 = await anotherGraph.invoke(/* ... */);
    return {
      result1,
      result2,
    };
  }
);
```

:::

??? example "扩展示例：从函数式 API 调用简单图"

    :::python
    ```python
    import uuid
    from typing import TypedDict
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph

    # 定义共享状态类型
    class State(TypedDict):
        foo: int

    # 定义一个简单的转换节点
    def double(state: State) -> State:
        return {"foo": state["foo"] * 2}

    # 使用图 API 构建图
    builder = StateGraph(State)
    builder.add_node("double", double)
    builder.set_entry_point("double")
    graph = builder.compile()

    # 定义函数式 API 流程
    checkpointer = InMemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(x: int) -> dict:
        result = graph.invoke({"foo": x})
        return {"bar": result["foo"]}

    # 执行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    print(workflow.invoke(5, config=config))  # 输出: {'bar': 10}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { entrypoint, MemorySaver } from "@langchain/langgraph";
    import { StateGraph } from "@langchain/langgraph";
    import { z } from "zod";

    // 定义共享状态类型
    const State = z.object({
      foo: z.number(),
    });

    // 使用图 API 构建图
    const builder = new StateGraph(State)
      .addNode("double", (state) => {
        return { foo: state.foo * 2 };
      })
      .addEdge("__start__", "double");
    const graph = builder.compile();

    // 定义函数式 API 流程
    const checkpointer = new MemorySaver();

    const workflow = entrypoint(
      { checkpointer, name: "workflow" },
      async (x: number) => {
        const result = await graph.invoke({ foo: x });
        return { bar: result.foo };
      }
    );

    // 执行流程
    const config = { configurable: { thread_id: uuidv4() } };
    console.log(await workflow.invoke(5, config)); // 输出: { bar: 10 }
    ```
    :::

## 调用其他入口点 (Entrypoints)

你可以在入口点 (entrypoint) 或任务 (task) 中调用其他入口点。

:::python

```python
@entrypoint() # 将自动使用父入口点的 checkpointer
def some_other_workflow(inputs: dict) -> int:
    return inputs["value"]

@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = some_other_workflow.invoke({"value": 1})
    return value
```

:::

:::js

```typescript
// 将自动使用父入口点的 checkpointer
const someOtherWorkflow = entrypoint(
  { name: "someOtherWorkflow" },
  async (inputs: { value: number }) => {
    return inputs.value;
  }
);

const myWorkflow = entrypoint(
  { checkpointer, name: "myWorkflow" },
  async (inputs: { value: number }) => {
    const value = await someOtherWorkflow.invoke({ value: 1 });
    return value;
  }
);
```

:::

??? example "扩展示例：调用另一个入口点"

    :::python
    ```python
    import uuid
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import InMemorySaver

    # 初始化一个 checkpointer
    checkpointer = InMemorySaver()

    # 一个可重用的子流程，用于计算乘积
    @entrypoint()
    def multiply(inputs: dict) -> int:
        return inputs["a"] * inputs["b"]

    # 调用子流程的主流程
    @entrypoint(checkpointer=checkpointer)
    def main(inputs: dict) -> dict:
        result = multiply.invoke({"a": inputs["x"], "b": inputs["y"]})
        return {"product": result}

    # 执行主流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    print(main.invoke({"x": 6, "y": 7}, config=config))  # 输出: {'product': 42}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import { entrypoint, MemorySaver } from "@langchain/langgraph";

    // 初始化一个 checkpointer
    const checkpointer = new MemorySaver();

    // 一个可重用的子流程，用于计算乘积
    const multiply = entrypoint(
      { name: "multiply" },
      async (inputs: { a: number; b: number }) => {
        return inputs.a * inputs.b;
      }
    );

    // 调用子流程的主流程
    const main = entrypoint(
      { checkpointer, name: "main" },
      async (inputs: { x: number; y: number }) => {
        const result = await multiply.invoke({ a: inputs.x, b: inputs.y });
        return { product: result };
      }
    );

    // 执行主流程
    const config = { configurable: { thread_id: uuidv4() } };
    console.log(await main.invoke({ x: 6, y: 7 }, config)); // 输出: { product: 42 }
    ```
    :::

## 流式输出 (Streaming)

**函数式 API** 使用与 **图 API** 相同的流式输出机制。有关更多详细信息，请阅读 [流式输出指南](../concepts/streaming.md) 部分。

使用 API 流式输出更新和自定义数据的示例。

:::python

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.config import get_stream_writer # (1)!

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs: dict) -> int:
    writer = get_stream_writer() # (2)!
    writer("Started processing") # (3)!
    result = inputs["x"] * 2
    writer(f"Result is {result}") # (4)!
    return result

config = {"configurable": {"thread_id": "abc"}}

# highlight-next-line
for mode, chunk in main.stream( # (5)!
    {"x": 5},
    stream_mode=["custom", "updates"], # (6)!
    config=config
):
    print(f"{mode}: {chunk}")
```

1. 从 `langgraph.config` 导入 `get_stream_writer`。
2. 在入口点内获取流式写入器实例。
3. 在计算开始前发出自定义数据。
4. 计算结果后发出另一条自定义消息。
5. 使用 `.stream()` 处理流式输出。
6. 指定要使用的流式输出模式。

```pycon
('updates', {'add_one': 2})
('updates', {'add_two': 3})
('custom', 'hello')
('custom', 'world')
('updates', {'main': 5})
```

!!! important "Python < 3.11 的异步操作"

    如果使用 Python < 3.11 且正在编写异步代码，`get_stream_writer()` 将无法正常工作。请改用 `StreamWriter` 类。有关更多详细信息，请参阅 [Python < 3.11 的异步操作](../how-tos/streaming.md#async)。

    ```python
    from langgraph.types import StreamWriter

    @entrypoint(checkpointer=checkpointer)
    # highlight-next-line
    async def main(inputs: dict, writer: StreamWriter) -> int:
        ...
    ```

:::

:::js

```typescript
import {
  entrypoint,
  MemorySaver,
  LangGraphRunnableConfig,
} from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const main = entrypoint(
  { checkpointer, name: "main" },
  async (
    inputs: { x: number },
    config: LangGraphRunnableConfig
  ): Promise<number> => {
    config.writer?.("Started processing"); // (1)!
    const result = inputs.x * 2;
    config.writer?.(`Result is ${result}`); // (2)!
    return result;
  }
);

const config = { configurable: { thread_id: "abc" } };

// (3)!
for await (const [mode, chunk] of await main.stream(
  { x: 5 },
  { streamMode: ["custom", "updates"], ...config } // (4)!
)) {
  console.log(`${mode}: ${JSON.stringify(chunk)}`);
}
```

1. 在计算开始前发出自定义数据。
2. 计算结果后发出另一条自定义消息。
3. 使用 `.stream()` 处理流式输出。
4. 指定要使用的流式输出模式。

```
updates: {"addOne": 2}
updates: {"addTwo": 3}
custom: "hello"
custom: "world"
updates: {"main": 5}
```

:::

## 重试策略

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy

# 此变量仅用于演示，以模拟网络故障。
# 在实际代码中不会用到。
attempts = 0

# 配置 RetryPolicy 以在发生 ValueError 时重试。
# 默认的 RetryPolicy 针对特定网络错误进行了优化。
retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry_policy=retry_policy)
def get_info():
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError('Failure')
    return "OK"

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer):
    return get_info().result()

config = {
    "configurable": {
        "thread_id": "1"
    }
}

main.invoke({'any_input': 'foobar'}, config=config)
```

```pycon
'OK'
```

:::

:::js

```typescript
import {
  MemorySaver,
  entrypoint,
  task,
  RetryPolicy,
} from "@langchain/langgraph";

// 此变量仅用于演示，以模拟网络故障。
// 在实际代码中不会用到。
let attempts = 0;

// 配置 RetryPolicy 以在发生 ValueError 时重试。
// 默认的 RetryPolicy 针对特定网络错误进行了优化。
const retryPolicy: RetryPolicy = { retryOn: (error) => error instanceof Error };

const getInfo = task(
  {
    name: "getInfo",
    retry: retryPolicy,
  },
  () => {
    attempts += 1;

    if (attempts < 2) {
      throw new Error("Failure");
    }
    return "OK";
  }
);

const checkpointer = new MemorySaver();

const main = entrypoint(
  { checkpointer, name: "main" },
  async (inputs: Record<string, any>) => {
    return await getInfo();
  }
);

const config = {
  configurable: {
    thread_id: "1",
  },
};

await main.invoke({ any_input: "foobar" }, config);
```

```
'OK'
```

:::

## 缓存任务 (Caching Tasks)

:::python

```python
import time
from langgraph.cache.memory import InMemoryCache
from langgraph.func import entrypoint, task
from langgraph.types import CachePolicy


@task(cache_policy=CachePolicy(ttl=120))  # (1)!
def slow_add(x: int) -> int:
    time.sleep(1)
    return x * 2


@entrypoint(cache=InMemoryCache())
def main(inputs: dict) -> dict[str, int]:
    result1 = slow_add(inputs["x"]).result()
    result2 = slow_add(inputs["x"]).result()
    return {"result1": result1, "result2": result2}


for chunk in main.stream({"x": 5}, stream_mode="updates"):
    print(chunk)

#> {'slow_add': 10}
#> {'slow_add': 10, '__metadata__': {'cached': True}}
#> {'main': {'result1': 10, 'result2': 10}}
```

1. `ttl` 以秒为单位指定。缓存将在该时间后失效。
   :::

:::js

```typescript
import {
  InMemoryCache,
  entrypoint,
  task,
  CachePolicy,
} from "@langchain/langgraph";

const slowAdd = task(
  {
    name: "slowAdd",
    cache: { ttl: 120 }, // (1)!
  },
  async (x: number) => {
    await new Promise((resolve) => setTimeout(resolve, 1000));
    return x * 2;
  }
);

const main = entrypoint(
  { cache: new InMemoryCache(), name: "main" },
  async (inputs: { x: number }) => {
    const result1 = await slowAdd(inputs.x);
    const result2 = await slowAdd(inputs.x);
    return { result1, result2 };
  }
);

for await (const chunk of await main.stream(
  { x: 5 },
  { streamMode: "updates" }
)) {
  console.log(chunk);
}

//> { slowAdd: 10 }
//> { slowAdd: 10, '__metadata__': { cached: true } }
//> { main: { result1: 10, result2: 10 } }
```

1. `ttl` 以秒为单位指定。缓存将在该时间后失效。
   :::

## 错误后恢复

:::python

```python
import time
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import StreamWriter

# 此变量仅用于演示，以模拟网络故障。
# 在实际代码中不会用到。
attempts = 0

@task()
def get_info():
    """
    模拟一个任务，该任务在成功之前会失败一次。
    首次尝试时引发异常，之后再返回 "OK"。
    """
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError("Failure")  # 模拟第一次尝试失败
    return "OK"

# 初始化一个内存中的 checkpointer 以进行持久化
checkpointer = InMemorySaver()

@task
def slow_task():
    """
    通过引入 1 秒延迟来模拟长时间运行的任务。
    """
    time.sleep(1)
    return "Ran slow task."

@entrypoint(checkpointer=checkpointer)
def main(inputs, writer: StreamWriter):
    """
    主流程函数，按顺序运行 slow_task 和 get_info 任务。

    参数:
    - inputs: 包含流程输入值的字典。
    - writer: 用于流式传输自定义数据的 StreamWriter。

    流程首先执行 `slow_task`，然后尝试执行 `get_info`，
    后者在第一次调用时会失败。
    """
    slow_task_result = slow_task().result()  # 阻塞调用 slow_task
    get_info().result()  # 第一次尝试时会在此处引发异常
    return slow_task_result

# 带有唯一 thread 标识符的流程执行配置
config = {
    "configurable": {
        "thread_id": "1"  # 用于跟踪流程执行的唯一标识符
    }
}

# 第一次调用将因 get_info 任务失败而引发异常
# 第一次调用会因 slow_task 执行而花费约 1 秒
try:
    main.invoke({'any_input': 'foobar'}, config=config)
except ValueError:
    pass  # 优雅地处理失败
```

当恢复执行时，由于 `slow_task` 的结果已保存在 checkpoint 中，因此无需重新运行它。

```python
main.invoke(None, config=config)
```

```pycon
'Ran slow task.'
```

:::

:::js

```typescript
import { entrypoint, task, MemorySaver } from "@langchain/langgraph";

// 此变量仅用于演示，以模拟网络故障。
// 在实际代码中不会用到。
let attempts = 0;

const getInfo = task("getInfo", async () => {
  /**
   * 模拟一个任务，该任务在成功之前会失败一次。
   * 首次尝试时引发异常，之后再返回 "OK"。
   */
  attempts += 1;

  if (attempts < 2) {
    throw new Error("Failure"); // 模拟第一次尝试失败
  }
  return "OK";
});

// 初始化一个内存中的 checkpointer 以进行持久化
const checkpointer = new MemorySaver();

const slowTask = task("slowTask", async () => {
  /**
   * 通过引入 1 秒延迟来模拟长时间运行的任务。
   */
  await new Promise((resolve) => setTimeout(resolve, 1000));
  return "Ran slow task.";
});

const main = entrypoint(
  { checkpointer, name: "main" },
  async (inputs: Record<string, any>) => {
    /**
     * 主流程函数，按顺序运行 slowTask 和 getInfo 任务。
     *
     * 参数:
     * - inputs: Record<string, any> 包含流程输入值。
     *
     * 流程首先执行 `slowTask`，然后尝试执行 `getInfo`，
     * 后者在第一次调用时会失败。
     */
    const slowTaskResult = await slowTask(); // 阻塞调用 slowTask
    await getInfo(); // 第一次尝试时会在此处引发异常
    return slowTaskResult;
  }
);

// 带有唯一 thread 标识符的流程执行配置
const config = {
  configurable: {
    thread_id: "1", // 用于跟踪流程执行的唯一标识符
  },
};

// 第一次调用将因 getInfo 任务失败而引发异常
// 第一次调用会因 slowTask 执行而花费约 1 秒
try {
  await main.invoke({ any_input: "foobar" }, config);
} catch (err) {
  // 优雅地处理失败
}
```

当恢复执行时，由于 `slowTask` 的结果已保存在 checkpoint 中，因此无需重新运行它。

```typescript
await main.invoke(null, config);
```

```
'Ran slow task.'
```

:::

## 人工干预 (Human-in-the-loop)

函数式 API 使用 `interrupt` 函数和 `Command` 原始类型来支持[人工干预](../concepts/human_in_the_loop.md)流程。

### 基本的人工干预流程

我们将创建三个[任务](../concepts/functional_api.md#task)：

1. 追加 "bar"。
2. 暂停等待人工输入。恢复时，追加人工输入。
3. 追加 "qux"。

:::python

```python
from langgraph.func import entrypoint, task
from langgraph.types import Command, interrupt


@task
def step_1(input_query):
    """追加 bar。"""
    return f"{input_query} bar"


@task
def human_feedback(input_query):
    """追加用户输入。"""
    feedback = interrupt(f"Please provide feedback: {input_query}")
    return f"{input_query} {feedback}"


@task
def step_3(input_query):
    """追加 qux。"""
    return f"{input_query} qux"
```

:::

:::js

```typescript
import { entrypoint, task, interrupt, Command } from "@langchain/langgraph";

const step1 = task("step1", async (inputQuery: string) => {
  // 追加 bar
  return `${inputQuery} bar`;
});

const humanFeedback = task("humanFeedback", async (inputQuery: string) => {
  // 追加用户输入
  const feedback = interrupt(`Please provide feedback: ${inputQuery}`);
  return `${inputQuery} ${feedback}`;
});

const step3 = task("step3", async (inputQuery: string) => {
  // 追加 qux
  return `${inputQuery} qux`;
});
```

:::

现在，我们可以在[入口点](../concepts/functional_api.md#entrypoint)中组合这些任务：

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def graph(input_query):
    result_1 = step_1(input_query).result()
    result_2 = human_feedback(result_1).result()
    result_3 = step_3(result_2).result()

    return result_3
```

:::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const graph = entrypoint(
  { checkpointer, name: "graph" },
  async (inputQuery: string) => {
    const result1 = await step1(inputQuery);
    const result2 = await humanFeedback(result1);
    const result3 = await step3(result2);

    return result3;
  }
);
```

:::

在任务中调用 [interrupt()](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt)，即可实现人工审阅和编辑上一个任务输出的功能。`step_1` 的结果会被持久化，因此在 `interrupt` 之后不会再次运行。

输入查询字符串：

:::python

```python
config = {"configurable": {"thread_id": "1"}}

for event in graph.stream("foo", config):
    print(event)
    print("\n")
```

:::

:::js

```typescript
const config = { configurable: { thread_id: "1" } };

for await (const event of await graph.stream("foo", config)) {
  console.log(event);
  console.log("\n");
}
```

:::

注意，我们在 `step_1` 之后使用 `interrupt` 暂停了。`interrupt` 提供了继续运行的说明。要继续，我们需要发出一个包含 `human_feedback` 任务所需数据的 [Command](../how-tos/human_in_the_loop/add-human-in-the-loop.md#resume-using-the-command-primitive)。

:::python

```python
# 继续执行
for event in graph.stream(Command(resume="baz"), config):
    print(event)
    print("\n")
```

:::

:::js

```typescript
// 继续执行
for await (const event of await graph.stream(
  new Command({ resume: "baz" }),
  config
)) {
  console.log(event);
  console.log("\n");
}
```

:::

恢复后，运行将继续执行剩余步骤并按预期终止。

### 审评工具调用

为了在执行前审阅工具调用，我们添加一个调用 [`interrupt`](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt) 的 `review_tool_call` 函数。当调用此函数时，执行将暂停，直到我们发出继续执行命令。

给定一个工具调用，我们的函数将暂停进行人工审阅。此时，我们可以：

- 接受工具调用
- 修改工具调用并继续
- 生成自定义工具消息（例如，指示模型重新格式化其工具调用）

:::python

```python
from typing import Union

def review_tool_call(tool_call: ToolCall) -> Union[ToolCall, ToolMessage]:
    """审阅工具调用，返回验证后的版本。"""
    human_review = interrupt(
        {
            "question": "Is this correct?",
            "tool_call": tool_call,
        }
    )
    review_action = human_review["action"]
    review_data = human_review.get("data")
    if review_action == "continue":
        return tool_call
    elif review_action == "update":
        updated_tool_call = {**tool_call, **{"args": review_data}}
        return updated_tool_call
    elif review_action == "feedback":
        return ToolMessage(
            content=review_data, name=tool_call["name"], tool_call_id=tool_call["id"]
        )
```

:::

:::js

```typescript
import { ToolCall } from "@langchain/core/messages/tool";
import { ToolMessage } from "@langchain/core/messages";

function reviewToolCall(toolCall: ToolCall): ToolCall | ToolMessage {
  // 审阅工具调用，返回验证后的版本
  const humanReview = interrupt({
    question: "Is this correct?",
    tool_call: toolCall,
  });

  const reviewAction = humanReview.action;
  const reviewData = humanReview.data;

  if (reviewAction === "continue") {
    return toolCall;
  } else if (reviewAction === "update") {
    const updatedToolCall = { ...toolCall, args: reviewData };
    return updatedToolCall;
  } else if (reviewAction === "feedback") {
    return new ToolMessage({
      content: reviewData,
      name: toolCall.name,
      tool_call_id: toolCall.id,
    });
  }

  throw new Error(`Unknown review action: ${reviewAction}`);
}
```

:::

现在，我们可以更新我们的[入口点](../concepts/functional_api.md#entrypoint)来审阅生成的工具调用。如果接受或修改了工具调用，执行方式与之前相同。否则，我们将仅追加由人工提供的 `ToolMessage`。先前任务的结果（即初始模型调用）会被持久化，因此在 `interrupt` 之后不会再次运行。

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph.message import add_messages
from langgraph.types import Command, interrupt


checkpointer = InMemorySaver()


@entrypoint(checkpointer=checkpointer)
def agent(messages, previous):
    if previous is not None:
        messages = add_messages(previous, messages)

    llm_response = call_model(messages).result()
    while True:
        if not llm_response.tool_calls:
            break

        # 审阅工具调用
        tool_results = []
        tool_calls = []
        for i, tool_call in enumerate(llm_response.tool_calls):
            review = review_tool_call(tool_call)
            if isinstance(review, ToolMessage):
                tool_results.append(review)
            else:  # 是一个验证过的工具调用
                tool_calls.append(review)
                if review != tool_call:
                    llm_response.tool_calls[i] = review  # 更新消息
        
        # 执行剩余的工具调用
        tool_result_futures = [call_tool(tool_call) for tool_call in tool_calls]
        remaining_tool_results = [fut.result() for fut in tool_result_futures]

        # 追加到消息列表
        messages = add_messages(
            messages,
            [llm_response, *tool_results, *remaining_tool_results],
        )

        # 再次调用模型
        llm_response = call_model(messages).result()

    # 生成最终响应
    messages = add_messages(messages, llm_response)
    return entrypoint.final(value=llm_response, save=messages)
```

:::

:::js

```typescript
import {
  MemorySaver,
  entrypoint,
  interrupt,
  Command,
  addMessages,
} from "@langchain/langgraph";
import { ToolMessage, AIMessage, BaseMessage } from "@langchain/core/messages";

const checkpointer = new MemorySaver();

const agent = entrypoint(
  { checkpointer, name: "agent" },
  async (
    messages: BaseMessage[],
    previous?: BaseMessage[]
  ): Promise<BaseMessage> => {
    if (previous !== undefined) {
      messages = addMessages(previous, messages);
    }

    let llmResponse = await callModel(messages);
    while (true) {
      if (!llmResponse.tool_calls?.length) {
        break;
      }

      // 审阅工具调用
      const toolResults: ToolMessage[] = [];
      const toolCalls: ToolCall[] = [];

      for (let i = 0; i < llmResponse.tool_calls.length; i++) {
        const review = reviewToolCall(llmResponse.tool_calls[i]);
        if (review instanceof ToolMessage) {
          toolResults.push(review);
        } else {
          // 是一个验证过的工具调用
          toolCalls.push(review);
          if (review !== llmResponse.tool_calls[i]) {
            llmResponse.tool_calls[i] = review; // 更新消息
          }
        }
      }

      // 执行剩余的工具调用
      const remainingToolResults = await Promise.all(
        toolCalls.map((toolCall) => callTool(toolCall))
      );

      // 追加到消息列表
      messages = addMessages(messages, [
        llmResponse,
        ...toolResults,
        ...remainingToolResults,
      ]);

      // 再次调用模型
      llmResponse = await callModel(messages);
    }

    // 生成最终响应
    messages = add_messages(messages, llmResponse);
    return entrypoint.final({ value: llmResponse, save: messages });
  }
);
```

:::

## 短期记忆 (Short-term memory)

短期记忆允许在同一[线程 ID](../concepts/functional_api.md#thread-id) 的不同[调用](../concepts/functional_api.md#invocation)之间存储信息。有关更多详细信息，请参阅[短期记忆](../concepts/functional_api.md#short-term-memory)。

### 管理 checkpoints

你可以查看和删除 checkpointer 存储的信息。

#### 查看线程状态 (checkpoint)

:::python

```python
config = {
    "configurable": {
        # highlight-next-line
        "thread_id": "1",
        # 可选地提供特定 checkpoint 的 ID，
        # 否则将显示最新的 checkpoint
        # highlight-next-line
        # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

    }
}
# highlight-next-line
graph.get_state(config)
```

```
StateSnapshot(
    values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
    metadata={
        'source': 'loop',
        'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
        'step': 4,
        'parents': {},
        'thread_id': '1'
    },
    created_at='2025-05-05T16:01:24.680462+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
    tasks=(),
    interrupts=()
)
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
    // 可选地提供特定 checkpoint 的 ID，
    // 否则将显示最新的 checkpoint
    // highlight-next-line
    // checkpoint_id: "1f029ca3-1f5b-6704-8004-820c16b69a5a"
  },
};
// highlight-next-line
await graph.getState(config);
```

```
StateSnapshot {
  values: {
    messages: [
      HumanMessage { content: "hi! I'm bob" },
      AIMessage { content: "Hi Bob! How are you doing today?" },
      HumanMessage { content: "what's my name?" },
      AIMessage { content: "Your name is Bob." }
    ]
  },
  next: [],
  config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
  metadata: {
    source: 'loop',
    writes: { call_model: { messages: AIMessage { content: "Your name is Bob." } } },
    step: 4,
    parents: {},
    thread_id: '1'
  },
  createdAt: '2025-05-05T16:01:24.680462+00:00',
  parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
  tasks: [],
  interrupts: []
}
```

:::

#### 查看线程历史记录 (checkpoints)

:::python

```python
config = {
    "configurable": {
        # highlight-next-line
        "thread_id": "1"
    }
}
# highlight-next-line
list(graph.get_state_history(config))
```

```
[
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
        next=('call_model',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.863421+00:00',
        parent_config={...}
        tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
        next=('__start__',),
        config={...},
        metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.863173+00:00',
        parent_config={...}
        tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
        next=(),
        config={...},
        metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:23.862295+00:00',
        parent_config={...}
        tasks=(),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob")]},
        next=('call_model',),
        config={...},
        metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:22.278960+00:00',
        parent_config={...}
        tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
        interrupts=()
    ),
    StateSnapshot(
        values={'messages': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
        metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
        created_at='2025-05-05T16:01:22.277497+00:00',
        parent_config=None,
        tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
        interrupts=()
    )
]
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
  },
};
// highlight-next-line
const history = [];
for await (const state of graph.getStateHistory(config)) {
  history.push(state);
}
```

```
[
  StateSnapshot {
    values: {
      messages: [
        HumanMessage { content: "hi! I'm bob" },
        AIMessage { content: "Hi Bob! How are you doing today? Is there anything I can help you with?" },
        HumanMessage { content: "what's my name?" },
        AIMessage { content: "Your name is Bob." }
      ]
    },
    next: [],
    config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
    metadata: { source: 'loop', writes: { call_model: { messages: AIMessage { content: "Your name is Bob." } } }, step: 4, parents: {}, thread_id: '1' },
    createdAt: '2025-05-05T16:01:24.680462+00:00',
    parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
    tasks: [],
    interrupts: []
  },
  // ... more state snapshots
]
```

:::

### 分离返回值和保存值

使用 `entrypoint.final` 来分离返回值给调用者和保存在 checkpoint 中的内容。这在以下情况很有用：

- 当你想返回一个计算结果（例如，摘要或状态），但将不同的内部值保存到 checkpoint 以供下次调用使用时。
- 当你需要控制传递给下一次运行的 `previous` 参数时。

:::python

```python
from typing import Optional
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import InMemorySaver

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: Optional[int]) -> entrypoint.final[int, int]:
    previous = previous or 0
    total = previous + n
    # 向调用者返回*先前*的值，但将*新的*总和保存到 checkpoint。
    return entrypoint.final(value=previous, save=total)

config = {"configurable": {"thread_id": "my-thread"}}

print(accumulate.invoke(1, config=config))  # 0
print(accumulate.invoke(2, config=config))  # 1
print(accumulate.invoke(3, config=config))  # 3
```

:::

:::js

```typescript
import { entrypoint, MemorySaver } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const accumulate = entrypoint(
  { checkpointer, name: "accumulate" },
  async (n: number, previous?: number) => {
    const prev = previous || 0;
    const total = prev + n;
    // 向调用者返回*先前*的值，但将*新的*总和保存到 checkpoint。
    return entrypoint.final({ value: prev, save: total });
  }
);

const config = { configurable: { thread_id: "my-thread" } };

console.log(await accumulate.invoke(1, config)); // 0
console.log(await accumulate.invoke(2, config)); // 1
console.log(await accumulate.invoke(3, config)); // 3
```

:::

### 聊天机器人示例

使用函数式 API 和 `InMemorySaver` checkpointer 的简单聊天机器人示例。
机器人能够记住之前的对话并从中断处继续。

:::python

```python
from langchain_core.messages import BaseMessage
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import InMemorySaver
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-3-5-sonnet-latest")

@task
def call_model(messages: list[BaseMessage]):
    response = model.invoke(messages)
    return response

checkpointer = InMemorySaver()

@entrypoint(checkpointer=checkpointer)
def workflow(inputs: list[BaseMessage], *, previous: list[BaseMessage]):
    if previous:
        inputs = add_messages(previous, inputs)

    response = call_model(inputs).result()
    return entrypoint.final(value=response, save=add_messages(inputs, response))

config = {"configurable": {"thread_id": "1"}}
input_message = {"role": "user", "content": "hi! I'm bob"}
for chunk in workflow.stream([input_message], config, stream_mode="values"):
    chunk.pretty_print()

input_message = {"role": "user", "content": "what's my name?"}
for chunk in workflow.stream([input_message], config, stream_mode="values"):
    chunk.pretty_print()
```

:::

:::js

```typescript
import { BaseMessage } from "@langchain/core/messages";
import {
  addMessages,
  entrypoint,
  task,
  MemorySaver,
} from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";

const model = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });

const callModel = task(
  "callModel",
  async (messages: BaseMessage[]): Promise<BaseMessage> => {
    const response = await model.invoke(messages);
    return response;
  }
);

const checkpointer = new MemorySaver();

const workflow = entrypoint(
  { checkpointer, name: "workflow" },
  async (
    inputs: BaseMessage[],
    previous?: BaseMessage[]
  ): Promise<BaseMessage> => {
    let messages = inputs;
    if (previous) {
      messages = addMessages(previous, inputs);
    }

    const response = await callModel(messages);
    return entrypoint.final({
      value: response,
      save: addMessages(messages, response),
    });
  }
);

const config = { configurable: { thread_id: "1" } };
const inputMessage = { role: "user", content: "hi! I'm bob" };

for await (const chunk of await workflow.stream([inputMessage], {
  ...config,
  streamMode: "values",
})) {
  console.log(chunk.content);
}

const inputMessage2 = { role: "user", content: "what's my name?" };
for await (const chunk of await workflow.stream([inputMessage2], {
  ...config,
  streamMode: "values",
})) {
  console.log(chunk.content);
}
```

:::

??? example "扩展示例：构建简单聊天机器人"

     [如何为函数式 API 添加线程级持久化](./persistence-functional.ipynb)：展示了如何为函数式 API 流程添加线程级持久化，并实现了一个简单的聊天机器人。

## 长期记忆 (Long-term memory)

[长期记忆](../concepts/memory.md#long-term-memory) 允许跨不同的**线程 ID** 存储信息。这对于学习一个用户在一场对话中的信息，并在另一场对话中使用这些信息可能很有用。

??? example "扩展示例：添加长期记忆"

    [如何为函数式 API 添加跨线程持久化](./cross-thread-persistence-functional.ipynb)：展示了如何为函数式 API 流程添加跨线程持久化，并实现了一个简单的聊天机器人。

## 流程 (Workflows)

- [流程和代理](../tutorials/workflows.md) 指南提供了更多关于如何使用函数式 API 构建流程的示例。

## 代理 (Agents)

- [如何从头开始创建代理（函数式 API）](./react-agent-from-scratch-functional.ipynb)：展示了如何使用函数式 API 从头开始创建一个简单的代理。
- [如何构建多代理网络](./multi-agent-network-functional.ipynb)：展示了如何使用函数式 API 构建多代理网络。
- [如何在多代理应用程序中添加多轮对话（函数式 API）](./multi-agent-multi-turn-convo-functional.ipynb)：允许最终用户与一个或多个代理进行多轮对话。

## 与其他库集成

- [使用函数式 API 将 LangGraph 的功能添加到其他框架](./autogen-integration-functional.ipynb)：将 LangGraph 的功能（如持久化、记忆和流式输出）添加到不自带这些功能的其他代理框架中。