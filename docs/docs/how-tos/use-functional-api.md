# 使用函数式 API

[**函数式 API**](../concepts/functional_api.md) 允许您以最少的代码更改，将 LangGraph 的核心功能—[持久化](../concepts/persistence.md)、[记忆](../how-tos/memory/add-memory.md)、[人工介入](../concepts/human_in_the_loop.md) 和[流式传输](../concepts/streaming.md)—添加到您的应用程序中。

!!! tip

    有关函数式 API 的概念信息，请参阅[函数式 API](../concepts/functional_api.md)。


## 创建一个简单的流程

在定义 `entrypoint` 时，输入仅限于函数的第一个参数。要传递多个输入，您可以使用字典。

```python
@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = inputs["value"]
    another_value = inputs["another_value"]
    ...

my_workflow.invoke({"value": 1, "another_value": 2})  
```

??? example "扩展示例：简单的流程"

    ```python
    import uuid
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import MemorySaver

    # 检查数字是否为偶数的任务
    @task
    def is_even(number: int) -> bool:
        return number % 2 == 0

    # 格式化消息的任务
    @task
    def format_message(is_even: bool) -> str:
        return "The number is even." if is_even else "The number is odd."

    # 创建一个用于持久化的检查点
    checkpointer = MemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(inputs: dict) -> str:
        """一个简单的流程，用于对数字进行分类。"""
        even = is_even(inputs["number"]).result()
        return format_message(even).result()

    # 使用唯一的线程 ID 运行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    result = workflow.invoke({"number": 7}, config=config)
    print(result)
    ```

??? example "扩展示例：用大语言模型写文章"

    本示例演示如何语法上使用 `@task` 和 `@entrypoint` 装饰器。鉴于提供了检查点，流程结果将持久化到检查点。

    ```python
    import uuid
    from langchain.chat_models import init_chat_model
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import MemorySaver

    llm = init_chat_model('openai:gpt-3.5-turbo')

    # 任务：使用大语言模型生成文章
    @task
    def compose_essay(topic: str) -> str:
        """针对给定主题生成一篇文章。"""
        return llm.invoke([
            {"role": "system", "content": "You are a helpful assistant that writes essays."},
            {"role": "user", "content": f"Write an essay about {topic}."}
        ]).content

    # 创建一个用于持久化的检查点
    checkpointer = MemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(topic: str) -> str:
        """一个简单的流程，用大语言模型生成文章。"""
        return compose_essay(topic).result()

    # 执行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    result = workflow.invoke("the history of flight", config=config)
    print(result)
    ```

## 并行执行

可以通过并发调用任务并等待结果来并行执行任务。这对于提高 I/O 密集型任务（例如调用大语言模型 API）的性能非常有用。

```python
@task
def add_one(number: int) -> int:
    return number + 1

@entrypoint(checkpointer=checkpointer)
def graph(numbers: list[int]) -> list[str]:
    futures = [add_one(i) for i in numbers]
    return [f.result() for f in futures]
```


??? example "扩展示例：并行调用大语言模型"

    本示例演示如何使用 `@task` 并行运行多个大语言模型调用。每次调用会针对不同主题生成一个段落，并将结果合并为一个文本输出。

    ```python
    import uuid
    from langchain.chat_models import init_chat_model
    from langgraph.func import entrypoint, task
    from langgraph.checkpoint.memory import MemorySaver

    # 初始化大语言模型
    llm = init_chat_model("openai:gpt-3.5-turbo")

    # 生成给定主题段落的任务
    @task
    def generate_paragraph(topic: str) -> str:
        response = llm.invoke([
            {"role": "system", "content": "You are a helpful assistant that writes educational paragraphs."},
            {"role": "user", "content": f"Write a paragraph about {topic}."}
        ])
        return response.content

    # 创建一个用于持久化的检查点
    checkpointer = MemorySaver()

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

    此示例利用 LangGraph 的并发模型来提高执行时间，尤其是在任务涉及大语言模型完成等 I/O 操作时。

## 调用图

函数式 API 和[图 API](../concepts/low_level.md) 可以在同一应用程序中协同使用，因为它们共享相同的底层运行时。

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

??? example "扩展示例：从函数式 API 调用简单图"

    ```python
    import uuid
    from typing import TypedDict
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import MemorySaver
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
    checkpointer = MemorySaver()

    @entrypoint(checkpointer=checkpointer)
    def workflow(x: int) -> dict:
        result = graph.invoke({"foo": x})
        return {"bar": result["foo"]}

    # 执行流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    print(workflow.invoke(5, config=config))  # 输出: {'bar': 10}
    ```


## 调用其他入口点

您可以在 `entrypoint` 或 `task` 中调用其他 `entrypoint`。

```python
@entrypoint() # 将自动使用父入口点的检查点
def some_other_workflow(inputs: dict) -> int:
    return inputs["value"]

@entrypoint(checkpointer=checkpointer)
def my_workflow(inputs: dict) -> int:
    value = some_other_workflow.invoke({"value": 1})
    return value
```

??? example "扩展示例：调用另一个入口点"

    ```python
    import uuid
    from langgraph.func import entrypoint
    from langgraph.checkpoint.memory import MemorySaver

    # 初始化一个检查点
    checkpointer = MemorySaver()

    # 一个可重用的子流程，用于乘以一个数字
    @entrypoint()
    def multiply(inputs: dict) -> int:
        return inputs["a"] * inputs["b"]

    # 主流程，用于调用子流程
    @entrypoint(checkpointer=checkpointer)
    def main(inputs: dict) -> dict:
        result = multiply.invoke({"a": inputs["x"], "b": inputs["y"]})
        return {"product": result}

    # 执行主流程
    config = {"configurable": {"thread_id": str(uuid.uuid4())}}
    print(main.invoke({"x": 6, "y": 7}, config=config))  # 输出: {'product': 42}
    ```
        

## 流式传输

函数式 API 使用与图 API 相同的流式传输机制。有关更多详细信息，请阅读[流式传输指南](../concepts/streaming.md#streaming)部分。

使用流式传输 API 来流式传输更新和自定义数据的示例。

```python
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import MemorySaver
from langgraph.config import get_stream_writer # (1)!

checkpointer = MemorySaver()

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
2. 在入口点内获取流写入器实例。
3. 在计算开始之前发出自定义数据。
4. 在计算结果后发出另一个自定义消息。
5. 使用 `.stream()` 处理流式输出。
6. 指定要使用的流模式。

```pycon
('updates', {'add_one': 2})
('updates', {'add_two': 3})
('custom', 'hello')
('custom', 'world')
('updates', {'main': 5})
```



!!! important "Python < 3.11 的异步支持"

    如果在 Python < 3.11 中使用并编写异步代码，则 `get_stream_writer()` 将不起作用。请改用 `StreamWriter` 类。有关更多详细信息，请参阅[Python < 3.11 的异步支持](../how-tos/streaming.md#async)。

    ```python
    from langgraph.types import StreamWriter

    @entrypoint(checkpointer=checkpointer)
    # highlight-next-line
    async def main(inputs: dict, writer: StreamWriter) -> int:
        ...
    ```

## 重试策略

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import RetryPolicy

# 此变量仅用于演示目的，以模拟网络故障。
# 它不是您实际代码中会有的东西。
attempts = 0

# 配置 RetryPolicy 以在 ValueError 时重试。
# 默认的 RetryPolicy 针对重试特定网络错误进行了优化。
retry_policy = RetryPolicy(retry_on=ValueError)

@task(retry_policy=retry_policy) 
def get_info():
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError('Failure')
    return "OK"

checkpointer = MemorySaver()

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

## 缓存任务

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

1. `ttl` 以秒为单位指定。缓存将在之后失效。

## 错误后恢复

```python
import time
from langgraph.checkpoint.memory import MemorySaver
from langgraph.func import entrypoint, task
from langgraph.types import StreamWriter

# 此变量仅用于演示目的，以模拟网络故障。
# 它不是您实际代码中会有的东西。
attempts = 0

@task()
def get_info():
    """
    模拟一个只会失败一次后成功的任务。
    首次尝试时引发异常，然后后续尝试返回“OK”。
    """
    global attempts
    attempts += 1

    if attempts < 2:
        raise ValueError("Failure")  # 模拟第一次尝试失败
    return "OK"

# 初始化内存检查点以进行持久化
checkpointer = MemorySaver()

@task
def slow_task():
    """
    通过引入 1 秒延迟来模拟一个运行缓慢的任务。
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
    后者将在第一次调用时引发异常。
    """
    slow_task_result = slow_task().result()  # 阻塞调用 slow_task
    get_info().result()  # 第一次调用时会在此处引发异常
    return slow_task_result

# 具有唯一线程标识符的流程执行配置
config = {
    "configurable": {
        "thread_id": "1"  # 用于跟踪流程执行的唯一标识符
    }
}

# 此调用将由于 slow_task 执行而耗时约 1 秒
try:
    # 第一次调用将因 get_info 任务失败而引发异常
    main.invoke({'any_input': 'foobar'}, config=config)
except ValueError:
    pass  # 优雅地处理失败
```

当恢复执行时，由于 `slow_task` 的结果已保存在检查点中，因此无需重新运行它。

```python
main.invoke(None, config=config)
```

```pycon
'Ran slow task.'
```

## 人工介入

函数式 API 使用 `interrupt` 函数和 `Command` 原语[支持人工介入](../concepts/human_in_the_loop.md)流程。

### 基本人工介入流程

我们将创建三个[任务](../concepts/functional_api.md#task)：

1. 追加 "bar"。
2. 暂停等待人工输入。恢复时，追加人工输入。
3. 追加 "qux"。

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

现在，我们可以在 `entrypoint` 中组合这些任务：

```python
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()


@entrypoint(checkpointer=checkpointer)
def graph(input_query):
    result_1 = step_1(input_query).result()
    result_2 = human_feedback(result_1).result()
    result_3 = step_3(result_2).result()

    return result_3
```

在任务内部调用 [interrupt()](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt) 会暂停执行，以便人工审查和编辑上一个任务的输出。先前任务的结果（在此例中为 `step_1`）会被持久化，因此在 `interrupt` 之后不会再次运行。

让我们输入一个查询字符串：

```python
config = {"configurable": {"thread_id": "1"}}

for event in graph.stream("foo", config):
    print(event)
    print("\n")
```

请注意，我们在 `step_1` 之后使用 `interrupt` 暂停了。`interrupt` 提供了恢复运行的说明。要恢复，我们发出一个包含 `human_feedback` 任务所需数据的 [Command](../how-tos/human_in_the_loop/add-human-in-the-loop.md#resume-using-the-command-primitive)。

```python
# 继续执行
for event in graph.stream(Command(resume="baz"), config):
    print(event)
    print("\n")
```
恢复后，运行将继续执行剩余的步骤，并按预期终止。

### 审核工具调用

要审核工具调用，我们添加一个 `review_tool_call` 函数，该函数调用 [`interrupt`](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt)。调用此函数时，执行将暂停，直到我们发出 resume 命令。

给定一个工具调用，我们的函数将为人工审核而 `interrupt`。此时我们可以：

- 接受工具调用
- 修改工具调用并继续
- 生成自定义工具消息（例如，指示模型重新格式化其工具调用）

```python
from typing import Union

def review_tool_call(tool_call: ToolCall) -> Union[ToolCall, ToolMessage]:
    """审核工具调用，返回一个已验证的版本。"""
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

现在我们可以更新我们的 `entrypoint` 以审核生成的工具调用。如果接受或修改了工具调用，我们将像以前一样执行。否则，我们只需追加人工提供的 `ToolMessage`。先前任务的结果（在此例中为初始模型调用）会被持久化，因此在 `interrupt` 之后不会再次运行。

```python
from langgraph.checkpoint.memory import MemorySaver
from langgraph.graph.message import add_messages
from langgraph.types import Command, interrupt


checkpointer = MemorySaver()


@entrypoint(checkpointer=checkpointer)
def agent(messages, previous):
    if previous is not None:
        messages = add_messages(previous, messages)

    llm_response = call_model(messages).result()
    while True:
        if not llm_response.tool_calls:
            break

        # 审核工具调用
        tool_results = []
        tool_calls = []
        for i, tool_call in enumerate(llm_response.tool_calls):
            review = review_tool_call(tool_call)
            if isinstance(review, ToolMessage):
                tool_results.append(review)
            else:  # 是验证过的工具调用
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

## 短期记忆

短期记忆允许在同一*线程 ID* 的不同*调用*之间存储信息。有关更多详细信息，请参阅[短期记忆](../concepts/functional_api.md#short-term-memory)。

### 管理检查点

您可以查看和删除检查点存储的信息。

#### 查看线程状态（检查点）

```python
config = {
    "configurable": {
        # highlight-next-line
        "thread_id": "1",
        # 可选地提供特定检查点的 ID，
        # 否则将显示最新的检查点
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

#### 查看线程历史记录（检查点）

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
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]}, 
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

### 分离返回值与已保存值

使用 `entrypoint.final` 来分离返回给调用者的内容与保存在检查点中的内容。这在以下情况很有用：

* 您想返回一个计算结果（例如，摘要或状态），但为下一次调用保存不同的内部值。
* 您需要控制传递给下一个运行的 `previous` 参数的内容。

```python
from typing import Optional
from langgraph.func import entrypoint
from langgraph.checkpoint.memory import MemorySaver

checkpointer = MemorySaver()

@entrypoint(checkpointer=checkpointer)
def accumulate(n: int, *, previous: Optional[int]) -> entrypoint.final[int, int]:
    previous = previous or 0
    total = previous + n
    # 返回 *上一个* 值给调用者，但将 *新* 总计保存到检查点。
    return entrypoint.final(value=previous, save=total)

config = {"configurable": {"thread_id": "my-thread"}}

print(accumulate.invoke(1, config=config))  # 0
print(accumulate.invoke(2, config=config))  # 1
print(accumulate.invoke(3, config=config))  # 3
```

### 聊天机器人示例

使用函数式 API 和 `MemorySaver` 检查点的简单聊天机器人示例。
机器人能够记住之前的对话并从中断的地方继续。

```python
from langchain_core.messages import BaseMessage
from langgraph.graph import add_messages
from langgraph.func import entrypoint, task
from langgraph.checkpoint.memory import MemorySaver
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(model="claude-3-5-sonnet-latest")

@task
def call_model(messages: list[BaseMessage]):
    response = model.invoke(messages)
    return response

checkpointer = MemorySaver()

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

??? example "扩展示例：构建一个简单的聊天机器人"

     [如何为函数式 API 添加线程级持久化](./persistence-functional.ipynb)：展示了如何为函数式 API 流程添加线程级持久化，并实现了一个简单的聊天机器人。

## 长期记忆

[长期记忆](../concepts/memory.md#long-term-memory)允许在不同的*线程 ID* 之间存储信息。这对于在一场对话中学习关于某个用户的信息，并在另一场对话中使用很有用。


??? example "扩展示例：添加长期记忆"

    [如何为函数式 API 添加跨线程持久化](./cross-thread-persistence-functional.ipynb)：展示了如何为函数式 API 流程添加跨线程持久化，并实现了一个简单的聊天机器人。

## 工作流

* [工作流和代理](../tutorials/workflows.md)指南提供了更多使用函数式 API 构建工作流的示例。

## 代理

* [如何从头开始创建代理（函数式 API）](./react-agent-from-scratch-functional.ipynb)：展示了如何使用函数式 API 从头开始创建一个简单的代理。
* [如何构建多代理网络](./multi-agent-network-functional.ipynb)：展示了如何使用函数式 API 构建多代理网络。
* [如何在多代理应用程序中添加多轮对话（函数式 API）](./multi-agent-multi-turn-convo-functional.ipynb)：允许最终用户与一个或多个代理进行多轮对话。  

## 与其他库集成

* [使用函数式 API 将 LangGraph 的功能添加到其他框架](./autogen-integration-functional.ipynb)：将 LangGraph 的持久化、记忆和流式传输等功能添加到不提供这些功能的其他代理框架中。