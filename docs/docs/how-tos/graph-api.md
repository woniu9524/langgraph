# 如何使用 Graph API

本指南演示了 LangGraph 的 Graph API 的基础知识。它将介绍如何[定义和更新状态](#define-and-update-state)，以及如何构建常见的图结构，例如[步骤序列](#create-a-sequence-of-steps)、[分支](#create-branches)和[循环](#create-and-control-loops)。它还涵盖了 LangGraph 的控制功能，包括用于 map-reduce 工作流的 [Send API](#map-reduce-and-the-send-api) 以及用于将状态更新与节点之间的“跳转”结合使用的 [Command API](#combine-control-flow-and-state-updates-with-command)。

## 设置

安装 `langgraph`：

```bash
pip install -U langgraph
```

!!! tip "设置 LangSmith 以获得更好的调试体验"
    注册 [LangSmith](https://smith.langchain.com) 以快速发现问题并提高 LangGraph 项目的性能。LangSmith 可让您利用跟踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用 — 在[文档](https://docs.smith.langchain.com)中阅读有关入门的更多信息。

## 定义和更新状态

在这里，我们展示了如何在 LangGraph 中定义和更新[状态](../concepts/low_level.md#state)。我们将演示：

1. 如何使用状态来定义图的[模式](../concepts/low_level.md#schema)
2. 如何使用[缩减器](../concepts/low_level.md#reducers)来控制状态更新的处理方式。

### 定义状态

LangGraph 中的[状态](../concepts/low_level.md#state)可以是 `TypedDict`、`Pydantic` 模型或数据类。下面我们将使用 `TypedDict`。有关使用 Pydantic 的详细信息，请参阅[此部分](#use-pydantic-models-for-graph-state)。

默认情况下，图将具有相同的输入和输出模式，状态决定了该模式。有关如何定义不同的输入和输出模式，请参阅[此部分](#define-input-and-output-schemas)。

让我们看一个使用[消息](../concepts/low_level.md#messagesstate)的简单示例。这代表了许多 LLM 应用的状态的一种通用形式。有关更多详细信息，请参阅我们的[概念页面](../concepts/low_level.md#working-with-messages-in-graph-state)。

```python
from langchain_core.messages import AnyMessage
from typing_extensions import TypedDict

class State(TypedDict):
    messages: list[AnyMessage]
    extra_field: int
```

此状态跟踪一个[消息](https://python.langchain.com/docs/concepts/messages/)对象列表以及一个额外的整数字段。

### 更新状态

让我们使用单个节点构建一个示例图。我们的[节点](../concepts/low_level.md#nodes)只是一个读取图状态并对其进行更新的 Python 函数。此函数的第一​​个参数始终是状态：

```python
from langchain_core.messages import AIMessage

def node(state: State):
    messages = state["messages"]
    new_message = AIMessage("Hello!")
    return {"messages": messages + [new_message], "extra_field": 10}
```

此节点只是将一条消息追加到我们的消息列表中，并填充一个额外的字段。

!!! important
    节点应直接返回对状态的更新，而不是修改状态。

接下来，让我们定义一个包含此节点的简单图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义一个在该状态上运行的图。然后，我们使用 [add_node](../concepts/low_level.md#nodes) 来填充我们的图。

```python
from langgraph.graph import StateGraph

builder = StateGraph(State)
builder.add_node(node)
builder.set_entry_point("node")
graph = builder.compile()
```

LangGraph 提供了一个可视化图的内置实用程序。让我们检查一下我们的图。有关可视化功能的详细信息，请参阅[此部分](#visualize-your-graph)。

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![具有单个节点的简单图](assets/graph_api_image_1.png)

在这种情况下，我们的图只执行一个节点。让我们继续简单的调用：

```python
from langchain_core.messages import HumanMessage

result = graph.invoke({"messages": [HumanMessage("Hi")]})
result
```
```
{'messages': [HumanMessage(content='Hi'), AIMessage(content='Hello!')], 'extra_field': 10}
```

请注意：

- 我们通过更新状态的单个键来启动调用。
- 我们在调用结果中接收整个状态。

为了方便起见，我们经常通过漂亮的打印来检查[消息对象](https://python.langchain.com/docs/concepts/messages/)的内容：

```python
for message in result["messages"]:
    message.pretty_print()
```
```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

### 使用缩减器处理状态更新

状态中的每个键都可以有自己独立的[缩减器](../concepts/low_level.md#reducers)函数，该函数控制如何应用来自节点的更新。如果未明确指定缩减器函数，则假定对该键的所有更新都应覆盖它。

对于 `TypedDict` 状态模式，我们可以通过用缩减器函数注解状态的相应字段来定义缩减器。

在前面的示例中，我们的节点通过将消息追加到其中来更新状态中的“messages”键。在下面，我们将一个缩减器添加到此键，以便自动追加更新：

```python
from typing_extensions import Annotated

def add(left, right):
    """也可以从 `operator` 内置模块导入 `add`。"""
    return left + right

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add]
    extra_field: int
```

现在我们可以简化节点：

```python
def node(state: State):
    new_message = AIMessage("Hello!")
    # highlight-next-line
    return {"messages": [new_message], "extra_field": 10}
```
```python
from langgraph.graph import StateGraph, START
graph = StateGraph(State).add_node(node).add_edge(START, "node").compile()

result = graph.invoke({"messages": [HumanMessage("Hi")]})

for message in result["messages"]:
    message.pretty_print()
```
```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

#### MessagesState

实际上，在更新消息列表时还有其他考虑因素：

- 我们可能希望更新状态中的现有消息。
- 我们可能希望接受[消息格式](../concepts/low_level.md#using-messages-in-your-graph)的简写，例如[OpenAI 格式](https://python.langchain.com/docs/concepts/messages/#openai-format)。

LangGraph 包含一个内置缩减器 `add_messages`，可处理这些注意事项：

```python
from langgraph.graph.message import add_messages

class State(TypedDict):
    # highlight-next-line
    messages: Annotated[list[AnyMessage], add_messages]
    extra_field: int

def node(state: State):
    new_message = AIMessage("Hello!")
    return {"messages": [new_message], "extra_field": 10}

graph = StateGraph(State).add_node(node).set_entry_point("node").compile()
```

```python
# highlight-next-line
input_message = {"role": "user", "content": "Hi"}

result = graph.invoke({"messages": [input_message]})

for message in result["messages"]:
    message.pretty_print()
```
```
================================ Human Message ================================

Hi
================================== Ai Message ==================================

Hello!
```

这对于涉及[聊天模型](https://python.langchain.com/docs/concepts/chat_models/)的应用来说是一种通用的状态表示。LangGraph 包含一个预构建的 `MessagesState` 以方便使用，这样我们就可以拥有：

```python
from langgraph.graph import MessagesState

class State(MessagesState):
    extra_field: int
```

### 定义输入和输出模式

默认情况下，`StateGraph` 使用单个模式运行，并且所有节点都应使用该模式进行通信。但是，也可以为图定义不同的输入和输出模式。

指定不同模式时，仍将使用内部模式在节点之间进行通信。输入模式确保提供的输入与预期的结构匹配，而输出模式则过滤内部数据，仅根据定义的输出模式返回相关信息。

下面，我们将看到如何定义不同的输入和输出模式。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 定义输入模式
class InputState(TypedDict):
    question: str

# 定义输出模式
class OutputState(TypedDict):
    answer: str

# 定义总体模式，结合输入和输出
class OverallState(InputState, OutputState):
    pass

# 定义处理输入并生成答案的节点
def answer_node(state: InputState):
    # 示例答案和一个额外键
    return {"answer": "bye", "question": state["question"]}

# 使用指定的输入和输出模式构建图
builder = StateGraph(OverallState, input_schema=InputState, output_schema=OutputState)
builder.add_node(answer_node)  # 添加答案节点
builder.add_edge(START, "answer_node")  # 定义起始边
builder.add_edge("answer_node", END)  # 定义结束边
graph = builder.compile()  # 编译图

# 使用输入调用图并打印结果
print(graph.invoke({"question": "hi"}))
```
```
{'answer': 'bye'}
```

请注意，invoke 的输出仅包含输出模式。

### 在节点之间传递私有状态

在某些情况下，您可能希望节点交换对中间逻辑至关重要但不需要成为图主模式一部分的信息。此私有数据与图的整体输入/输出无关，应仅在某些节点之间共享。

下面，我们将创建一个由三个节点组成的简单顺序图（node_1、node_2 和 node_3），其中私有数据在前两个步骤（node_1 和 node_2）之间传递，而第三个步骤（node_3）只能访问公共整体状态。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict

# 图的整体状态（这是跨节点共享的公共状态）
class OverallState(TypedDict):
    a: str

# node_1 的输出包含不属于整体状态的私有数据
class Node1Output(TypedDict):
    private_data: str

# 私有数据仅在 node_1 和 node_2 之间共享
def node_1(state: OverallState) -> Node1Output:
    output = {"private_data": "set by node_1"}
    print(f"Entered node `node_1`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# node_2 的输入仅请求 node_1 之后可用的私有数据
class Node2Input(TypedDict):
    private_data: str

def node_2(state: Node2Input) -> OverallState:
    output = {"a": "set by node_2"}
    print(f"Entered node `node_2`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# node_3 只能访问整体状态（无法访问来自 node_1 的私有数据）
def node_3(state: OverallState) -> OverallState:
    output = {"a": "set by node_3"}
    print(f"Entered node `node_3`:\n\tInput: {state}.\n\tReturned: {output}")
    return output

# 按顺序连接节点
# node_2 接受来自 node_1 的私有数据，而
# node_3 不会看到私有数据。
builder = StateGraph(OverallState).add_sequence([node_1, node_2, node_3])
builder.add_edge(START, "node_1")
graph = builder.compile()

# 使用初始状态调用图
response = graph.invoke(
    {
        "a": "set at start",
    }
)

print()
print(f"Output of graph invocation: {response}")
```
```
Entered node `node_1`:
	Input: {'a': 'set at start'}.
	Returned: {'private_data': 'set by node_1'}
Entered node `node_2`:
	Input: {'private_data': 'set by node_1'}.
	Returned: {'a': 'set by node_2'}
Entered node `node_3`:
	Input: {'a': 'set by node_2'}.
	Returned: {'a': 'set by node_3'}

Output of graph invocation: {'a': 'set by node_3'}
```

### 使用 Pydantic 模型进行图状态管理

`StateGraph` 在初始化时接受 `state_schema` 参数，该参数指定了图中节点可以访问和更新的状态的“形状”。

在我们的示例中，我们通常为 `state_schema` 使用 Python 原生的 `TypedDict`，但 `state_schema` 可以是任何[类型](https://docs.python.org/3/library/stdtypes.html#type-objects)。

在这里，我们将看到如何使用[Pydantic BaseModel](https://docs.pydantic.dev/latest/api/base_model/) 作为 `state_schema` 来为**输入**添加运行时验证。

!!! note "已知限制"
    - 当前，图的输出**不会**是 pydantic 模型的实例。
    - 运行时验证仅发生在节点输入上，而不发生在输出上。
    - pydantic 的验证错误跟踪不显示错误源自哪个节点。

```python
from langgraph.graph import StateGraph, START, END
from typing_extensions import TypedDict
from pydantic import BaseModel

# 图的整体状态（这是跨节点共享的公共状态）
class OverallState(BaseModel):
    a: str

def node(state: OverallState):
    return {"a": "goodbye"}

# 构建状态图
builder = StateGraph(OverallState)
builder.add_node(node)  # node_1 是第一个节点
builder.add_edge(START, "node")  # 用 node_1 开始图
builder.add_edge("node", END)  # 在 node_1 之后结束图
graph = builder.compile()

# 使用有效输入测试图
graph.invoke({"a": "hello"})
```

使用**无效**输入调用图

```python
try:
    graph.invoke({"a": 123})  # 应为字符串
except Exception as e:
    print("引发了异常，因为 `a` 是整数而非字符串。")
    print(e)
```
```
引发了异常，因为 `a` 是整数而非字符串。
1 validation error for OverallState
a
  Input should be a valid string [type=string_type, input_value=123, input_type=int]
    For further information visit https://errors.pydantic.dev/2.9/v/string_type
```

有关 Pydantic 模型状态的其他功能，请参见下文：

??? example "序列化行为"

    当使用 Pydantic 模型作为状态模式时，了解序列化工作方式很重要，尤其是在以下情况下：
    - 将 Pydantic 对象作为输入传递
    - 从图接收输出
    - 处理嵌套的 Pydantic 模型

    让我们看看这些行为的实际应用。

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class NestedModel(BaseModel):
        value: str

    class ComplexState(BaseModel):
        text: str
        count: int
        nested: NestedModel

    def process_node(state: ComplexState):
        # 节点接收已验证的 Pydantic 对象
        print(f"Input state type: {type(state)}")
        print(f"Nested type: {type(state.nested)}")
        # 返回字典更新
        return {"text": state.text + " processed", "count": state.count + 1}

    # 构建图
    builder = StateGraph(ComplexState)
    builder.add_node("process", process_node)
    builder.add_edge(START, "process")
    builder.add_edge("process", END)
    graph = builder.compile()

    # 创建 Pydantic 实例以供输入
    input_state = ComplexState(text="hello", count=0, nested=NestedModel(value="test"))
    print(f"Input object type: {type(input_state)}")

    # 使用 Pydantic 实例调用图
    result = graph.invoke(input_state)
    print(f"Output type: {type(result)}")
    print(f"Output content: {result}")

    # 如果需要，转换回 Pydantic 模型
    output_model = ComplexState(**result)
    print(f"Converted back to Pydantic: {type(output_model)}")
    ```

??? example "运行时类型转换"

    Pydantic 会对某些数据类型执行运行时类型转换。这可能很有帮助，但如果您不了解它，也可能导致意外行为。

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel

    class CoercionExample(BaseModel):
        # Pydantic 会将字符串数字转换为整数
        number: int
        # Pydantic 会将字符串布尔值解析为 bool
        flag: bool

    def inspect_node(state: CoercionExample):
        print(f"number: {state.number} (type: {type(state.number)})")
        print(f"flag: {state.flag} (type: {type(state.flag)})")
        return {}

    builder = StateGraph(CoercionExample)
    builder.add_node("inspect", inspect_node)
    builder.add_edge(START, "inspect")
    builder.add_edge("inspect", END)
    graph = builder.compile()

    # 使用将要转换的字符串输入演示转换
    result = graph.invoke({"number": "42", "flag": "true"})

    # 这将因验证错误而失败
    try:
        graph.invoke({"number": "not-a-number", "flag": "true"})
    except Exception as e:
        print(f"\nExpected validation error: {e}")
    ```

??? example "处理消息模型"

    在状态模式中使用 LangChain 消息类型时，序列化有一些重要的注意事项。您应该使用 `AnyMessage`（而不是 `BaseMessage`）以便在通过网络传递消息对象时进行正确的序列化/反序列化。

    ```python
    from langgraph.graph import StateGraph, START, END
    from pydantic import BaseModel
    from langchain_core.messages import HumanMessage, AIMessage, AnyMessage
    from typing import List

    class ChatState(BaseModel):
        messages: List[AnyMessage]
        context: str

    def add_message(state: ChatState):
        return {"messages": state.messages + [AIMessage(content="Hello there!")]}

    builder = StateGraph(ChatState)
    builder.add_node("add_message", add_message)
    builder.add_edge(START, "add_message")
    builder.add_edge("add_message", END)
    graph = builder.compile()

    # 使用消息创建输入
    initial_state = ChatState(
        messages=[HumanMessage(content="Hi")], context="Customer support chat"
    )

    result = graph.invoke(initial_state)
    print(f"Output: {result}")

    # 转换回 Pydantic 模型以查看消息类型
    output_model = ChatState(**result)
    for i, msg in enumerate(output_model.messages):
        print(f"Message {i}: {type(msg).__name__} - {msg.content}")
    ```

## 添加运行时配置

有时您希望在调用图时能够配置它。例如，您可能希望能够在运行时指定使用哪个 LLM 或系统提示，*而不会污染图状态中的这些参数*。

要添加运行时配置：

1. 指定配置的模式
2. 将配置添加到节点的函数签名或条件边的函数签名中
3. 将配置传递到图中。

下面是一个简单的示例：

```python
from langchain_core.runnables import RunnableConfig
from langgraph.graph import END, StateGraph, START
from typing_extensions import TypedDict

# 1. 指定配置模式
class ConfigSchema(TypedDict):
    my_runtime_value: str

# 2. 定义一个在节点中访问配置的图
class State(TypedDict):
    my_state_value: str

# highlight-next-line
def node(state: State, config: RunnableConfig):
    # highlight-next-line
    if config["configurable"]["my_runtime_value"] == "a":
        return {"my_state_value": 1}
        # highlight-next-line
    elif config["configurable"]["my_runtime_value"] == "b":
        return {"my_state_value": 2}
    else:
        raise ValueError("Unknown values.")

# highlight-next-line
builder = StateGraph(State, config_schema=ConfigSchema)
builder.add_node(node)
builder.add_edge(START, "node")
builder.add_edge("node", END)

graph = builder.compile()

# 3. 在运行时传递配置：
# highlight-next-line
print(graph.invoke({}, {"configurable": {"my_runtime_value": "a"}}))
# highlight-next-line
print(graph.invoke({}, {"configurable": {"my_runtime_value": "b"}}))
```
```
{'my_state_value': 1}
{'my_state_value': 2}
```

??? example "扩展示例：在运行时指定 LLM"
    下面我们演示一个实际示例，其中我们在运行时配置要使用的 LLM。我们将使用 OpenAI 和 Anthropic 模型。

    ```python
    from langchain.chat_models import init_chat_model
    from langchain_core.runnables import RunnableConfig
    from langgraph.graph import MessagesState
    from langgraph.graph import END, StateGraph, START
    from typing_extensions import TypedDict

    class ConfigSchema(TypedDict):
        model: str

    MODELS = {
        "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
        "openai": init_chat_model("openai:gpt-4.1-mini"),
    }

    def call_model(state: MessagesState, config: RunnableConfig):
        model = config["configurable"].get("model", "anthropic")
        model = MODELS[model]
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    builder = StateGraph(MessagesState, config_schema=ConfigSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # 用途
    input_message = {"role": "user", "content": "hi"}
    # 不带配置，使用默认值（Anthropic）
    response_1 = graph.invoke({"messages": [input_message]})["messages"][-1]
    # 或者，可以设置 OpenAI
    config = {"configurable": {"model": "openai"}}
    response_2 = graph.invoke({"messages": [input_message]}, config=config)["messages"][-1]

    print(response_1.response_metadata["model_name"])
    print(response_2.response_metadata["model_name"])
    ```
    ```
    claude-3-5-haiku-20241022
    gpt-4.1-mini-2025-04-14
    ```

??? example "扩展示例：在运行时指定模型和系统消息"
    下面我们演示一个实际示例，其中我们在运行时配置两个参数：LLM 和系统消息。

    ```python
    from typing import Optional
    from langchain.chat_models import init_chat_model
    from langchain_core.messages import SystemMessage
    from langchain_core.runnables import RunnableConfig
    from langgraph.graph import END, MessagesState, StateGraph, START
    from typing_extensions import TypedDict

    class ConfigSchema(TypedDict):
        model: Optional[str]
        system_message: Optional[str]

    MODELS = {
        "anthropic": init_chat_model("anthropic:claude-3-5-haiku-latest"),
        "openai": init_chat_model("openai:gpt-4.1-mini"),
    }

    def call_model(state: MessagesState, config: RunnableConfig):
        model = config["configurable"].get("model", "anthropic")
        model = MODELS[model]
        messages = state["messages"]
        if system_message := config["configurable"].get("system_message"):
            messages = [SystemMessage(system_message)] + messages
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState, config_schema=ConfigSchema)
    builder.add_node("model", call_model)
    builder.add_edge(START, "model")
    builder.add_edge("model", END)

    graph = builder.compile()

    # 用途
    input_message = {"role": "user", "content": "hi"}
    config = {"configurable": {"model": "openai", "system_message": "Respond in Italian."}}
    response = graph.invoke({"messages": [input_message]}, config)
    for message in response["messages"]:
        message.pretty_print()
    ```
    ```
    ================================ Human Message ================================

    hi
    ================================== Ai Message ==================================

    Ciao! Come posso aiutarti oggi?
    ```

## 添加重试策略

在许多用例中，您可能希望您的节点具有自定义的重试策略，例如，当您调用 API、查询数据库或调用 LLM 等时。LangGraph 允许您为节点添加重试策略。

要配置重试策略，请将 `retry_policy` 参数传递给 [add_node](../reference/graphs.md#langgraph.graph.state.StateGraph.add_node)。 `retry_policy` 参数接受一个 ` RetryPolicy` 命名元组对象。下面我们使用默认参数实例化一个 `RetryPolicy` 对象并将其与节点关联：

```python
from langgraph.pregel import RetryPolicy

builder.add_node(
    "node_name",
    node_function,
    retry_policy=RetryPolicy(),
)
```

默认情况下，`retry_on` 参数使用 `default_retry_on` 函数，该函数会重试除以下以外的任何异常：

*   `ValueError`
*   `TypeError`
*   `ArithmeticError`
*   `ImportError`
*   `LookupError`
*   `NameError`
*   `SyntaxError`
*   `RuntimeError`
*   `ReferenceError`
*   `StopIteration`
*   `StopAsyncIteration`
*   `OSError`

此外，对于来自流行 HTTP 请求库（如 `requests` 和 `httpx`）的异常，它仅重试 5xx 状态代码。

??? example "扩展示例：自定义重试策略"
    考虑一个我们从 SQL 数据库读取的示例。下面我们向节点传递两个不同的重试策略：

    ```python
    import sqlite3
    from typing_extensions import TypedDict
    from langchain.chat_models import init_chat_model
    from langgraph.graph import END, MessagesState, StateGraph, START
    from langgraph.pregel import RetryPolicy
    from langchain_community.utilities import SQLDatabase
    from langchain_core.messages import AIMessage

    db = SQLDatabase.from_uri("sqlite:///:memory:")
    model = init_chat_model("anthropic:claude-3-5-haiku-latest")

    def query_database(state: MessagesState):
        query_result = db.run("SELECT * FROM Artist LIMIT 10;")
        return {"messages": [AIMessage(content=query_result)]}

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": [response]}

    # 定义一个新图
    builder = StateGraph(MessagesState)
    builder.add_node(
        "query_database",
        query_database,
        retry_policy=RetryPolicy(retry_on=sqlite3.OperationalError),
    )
    builder.add_node("model", call_model, retry_policy=RetryPolicy(max_attempts=5))
    builder.add_edge(START, "model")
    builder.add_edge("model", "query_database")
    builder.add_edge("query_database", END)
    graph = builder.compile()
    ```

## 添加节点缓存

节点缓存对于避免重复操作很有用，比如在执行某些开销大（在时间或成本方面）的操作时。LangGraph 允许您为图中的节点添加单独的缓存策略。

要配置缓存策略，请将 `cache_policy` 参数传递给 [add_node](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.state.StateGraph.add_node) 函数。在下面的示例中，`[`CachePolicy`](https://langchain-ai.github.io/langgraph/reference/types/?h=cachepolicy#langgraph.types.CachePolicy)` 对象使用 120 秒的生存时间 (ttl) 和默认的 `key_func` 生成器进行实例化。然后将其与节点关联：

```python
from langgraph.types import CachePolicy

builder.add_node(
    "node_name",
    node_function,
    cache_policy=CachePolicy(ttl=120),
)
```

然后，要为图启用节点级缓存，请在编译图时设置 `cache` 参数。下面的示例使用 `InMemoryCache` 来设置具有内存中缓存的图，但也提供了 `SqliteCache`。

```python
from langgraph.cache.memory import InMemoryCache

graph = builder.compile(cache=InMemoryCache())
```

## 创建步骤序列

!!! info "先决条件"
    本指南假定您熟悉上面关于[状态](#define-and-update-state)的部分。

这里我们演示如何构建一个简单的步骤序列。我们将展示：

1. 如何构建顺序图
2. 用于构建类似图的内置简写方式。

要添加节点序列，我们使用图的 `.add_node` 和 `.add_edge` 方法[图](../concepts/low_level.md#stategraph)：

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# 添加节点
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# 添加边
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```

我们也可以使用内置的简写 `.add_sequence`：

```python
builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
builder.add_edge(START, "step_1")
```

??? info "为什么将应用程序步骤分解为 LangGraph 序列？"
    LangGraph 可以轻松地为您的应用程序添加底层持久层。
    这允许在节点执行之间检查状态，因此您的 LangGraph 节点负责：

    - 如何[检查状态](../concepts/persistence.md)
    - 如何在[人工介入](../concepts/human_in_the_loop.md)工作流中恢复中断
    - 如何使用 LangGraph 的[时间旅行](../concepts/time-travel.md)功能“倒回”并分支执行

    它们还决定了如何[流式传输](../concepts/streaming.md)执行步骤，以及如何使用[LangGraph Studio](../concepts/langgraph_studio.md) 可视化和调试您的应用程序。

让我们演示一个端到端示例。我们将创建三个步骤的序列：

1. 在状态的键中填充一个值
2. 更新相同的值
3. 填充不同的值

首先，让我们定义我们的[状态](../concepts/low_level.md#state)。这会影响图的[模式](../concepts/low_level.md#schema) ，并且还可以指定如何应用更新。有关更多详细信息，请参阅[此部分](#process-state-updates-with-reducers)。

在我们的例子中，我们只跟踪两个值：

```python
from typing_extensions import TypedDict

class State(TypedDict):
    value_1: str
    value_2: int
```

我们的[节点](../concepts/low_level.md#nodes)只是读取图状态并对其进行更新的 Python 函数。此函数的第一​​个参数始终是状态：

```python
def step_1(state: State):
    return {"value_1": "a"}

def step_2(state: State):
    current_value_1 = state["value_1"]
    return {"value_1": f"{current_value_1} b"}

def step_3(state: State):
    return {"value_2": 10}
```

!!! note
    请注意，在发出状态更新时，每个节点都可以只指定它希望更新的键的值。

    默认情况下，这将**覆盖**相应键的值。您还可以使用[缩减器](../concepts/low_level.md#reducers)来控制如何处理更新——例如，您可以改用追加 successive 更新。有关如何使用缩减器更新状态的更多详细信息，请参阅[此部分](#process-state-updates-with-reducers)。

最后，我们定义图。我们使用 [StateGraph](../concepts/low_level.md#stategraph) 来定义一个在该状态上运行的图。

然后，我们将使用 [add_node](../concepts/low_level.md#messagesstate) 和 [add_edge](../concepts/low_level.md#edges) 来填充我们的图并定义其控制流。

```python
from langgraph.graph import START, StateGraph

builder = StateGraph(State)

# 添加节点
builder.add_node(step_1)
builder.add_node(step_2)
builder.add_node(step_3)

# 添加边
builder.add_edge(START, "step_1")
builder.add_edge("step_1", "step_2")
builder.add_edge("step_2", "step_3")
```

!!! tip "指定自定义名称"
    您可以使用 `.add_node` 为节点指定自定义名称：

    ```python
    builder.add_node("my_node", step_1)
    ```

请注意：

- `.add_edge` 接受节点名称，对于函数默认为 `node.__name__`。
- 我们必须指定图的入口点。为此，我们添加了一个带有[START 节点](../concepts/low_level.md#start-node)的边。
- 当没有更多节点要执行时，图将停止。

我们接下来[编译](../concepts/low_level.md#compiling-your-graph)我们的图。这会对图的结构进行一些基本检查（例如，识别孤立节点）。如果我们要通过[检查器](../concepts/persistence.md)为应用程序添加持久性，它也将在这里传递。

```python
graph = builder.compile()
```

LangGraph 提供了一个可视化图的内置实用程序。让我们检查我们的序列。有关可视化功能的详细信息，请参阅[本指南](#visualize-your-graph)。

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![步骤序列图](assets/graph_api_image_2.png)

让我们继续简单的调用：

```python
graph.invoke({"value_1": "c"})
```
```
{'value_1': 'a b', 'value_2': 10}
```

请注意：

- 我们通过提供一个状态键的值来启动调用。我们必须至少提供一个键的值。
- 我们传入的值被第一个节点覆盖。
- 第二个节点更新了该值。
- 第三个节点填充了不同的值。

!!! tip "内置简写"
    `langgraph>=0.2.46` 包含一个内置的简写 `add_sequence` 用于添加节点序列。您可以如下编译相同的图：

    ```python
    # highlight-next-line
    builder = StateGraph(State).add_sequence([step_1, step_2, step_3])
    builder.add_edge(START, "step_1")

    graph = builder.compile()

    graph.invoke({"value_1": "c"})
    ```

## 创建分支

节点并行执行对于加快整体图操作至关重要。LangGraph 为节点的并行执行提供原生支持，这可以显著提高基于图的工作流的性能。这种并行化通过扇出和扇入机制实现，利用标准边和[条件边](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.MessageGraph.add_conditional_edges)。下面是一些示例，展示了如何添加创建适合您的分支数据流。

### 并行运行图节点

在此示例中，我们从 `Node A` 扇出到 `B 和 C`，然后扇入到 `D`。通过我们的状态，[我们指定缩减器 add 操作](https://langchain-ai.github.io/langgraph/concepts/low_level.md#reducers)。这将组合或累加状态中特定键的值，而不是简单地覆盖现有值。对于列表，这意味着将新列表与现有列表连接。有关使用缩减器更新状态的更多详细信息，请参阅上面的[状态缩减器](#process-state-updates-with-reducers)部分。

```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add 缩减器函数使此为仅追加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_node(d)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![并行执行图](assets/graph_api_image_3.png)

使用缩减器，您可以看到每个节点中添加的值会被累加。

```python
graph.invoke({"aggregate": []}, {"configurable": {"thread_id": "foo"}})
```
```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "D" to ['A', 'B', 'C']
```

!!! note
    在上面的示例中，节点 `"b"` 和 `"c"` 在同一个[超级步骤](../concepts/low_level.md#graphs)中并行执行。因为它们在同一个步骤中，所以节点 `"d"` 在 `"b"` 和 `"c"` 都完成后执行。

    重要的是，并行超级步骤的更新可能不会保持一致的顺序。如果您需要并行超级步骤的更新的连续、预定顺序，您应该将输出与用于排序的值一起写入状态中的单独字段。

??? note "异常处理？"
    LangGraph 在[超级步骤](../concepts/low_level.md#graphs)内执行节点，这意味着虽然并行分支并行执行，但整个超级步骤是**事务性的**。如果其中任何分支引发异常，**所有**更新都不会应用于状态（整个超级步骤都会出错）。

    重要的是，在使用[检查器](../concepts/persistence.md)时，超级步骤中成功节点的 결과 会被保存，并在恢复时不会重复。

    如果您有易出错的操作（可能想要处理不稳定的 API 调用），LangGraph 提供两种方法来解决此问题：

    1. 可以在节点内编写常规的 Python 代码来捕获和处理异常。
    2. 可以设置**[重试策略](../reference/types.md#langgraph.types.RetryPolicy)** 来指示图重试引发某些类型异常的节点。只重试失败的分支，因此您无需担心执行冗余工作。

    两者结合起来，可以让您执行并行执行并完全控制异常处理。

### 延迟节点执行

当您希望将节点的执行延迟到所有其他挂起任务完成后时，延迟节点执行非常有用。这在分支长度不同时尤其重要，在类似 map-reduce 的工作流中很常见。

上面的示例展示了如何在每个路径仅为一个步骤时进行扇出和扇入。但如果一个分支有多个步骤呢？让我们在 `"b"` 分支中添加一个节点 `"b_2"`：

```python
import operator
from typing import Annotated, Any
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add 缩减器函数使此为仅追加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def b_2(state: State):
    print(f'Adding "B_2" to {state["aggregate"]}')
    return {"aggregate": ["B_2"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

def d(state: State):
    print(f'Adding "D" to {state["aggregate"]}')
    return {"aggregate": ["D"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(b_2)
builder.add_node(c)
# highlight-next-line
builder.add_node(d, defer=True)
builder.add_edge(START, "a")
builder.add_edge("a", "b")
builder.add_edge("a", "c")
builder.add_edge("b", "b_2")
builder.add_edge("b_2", "d")
builder.add_edge("c", "d")
builder.add_edge("d", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![延迟执行图](assets/graph_api_image_4.png)

```python
graph.invoke({"aggregate": []})
```
```
Adding "A" to []
Adding "B" to ['A']
Adding "C" to ['A']
Adding "B_2" to ['A', 'B', 'C']
Adding "D" to ['A', 'B', 'C', 'B_2']
```

在上面的示例中，节点 `"b"` 和 `"c"` 在同一个超级步骤中并行执行。我们将 `defer=True` 设置为节点 `d`，因此它将在所有挂起任务完成后执行。在这种情况下，这意味着 `"d"` 要等到整个 `"b"` 分支完成后才执行。

### 条件分支

如果您的扇出应根据状态在运行时变化，您可以使用[条件边](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.StateGraph.add_conditional_edges)来使用图状态选择一个或多个路径。请参阅下面的示例，其中节点 `a` 生成一个状态更新，该更新决定了下一个节点。

```python
import operator
from typing import Annotated, Literal, Sequence
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    aggregate: Annotated[list, operator.add]
    # 向状态添加一个键。我们将设置此键来确定
    # 如何分支。
    which: str

def a(state: State):
    print(f'Adding "A" to {state["aggregate"]}')
    # highlight-next-line
    return {"aggregate": ["A"], "which": "c"}

def b(state: State):
    print(f'Adding "B" to {state["aggregate"]}')
    return {"aggregate": ["B"]}

def c(state: State):
    print(f'Adding "C" to {state["aggregate"]}')
    return {"aggregate": ["C"]}

builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)
builder.add_node(c)
builder.add_edge(START, "a")
builder.add_edge("b", END)
builder.add_edge("c", END)

def conditional_edge(state: State) -> Literal["b", "c"]:
    # 在此处填充使用状态来确定下一个节点的任意逻辑
    return state["which"]

# highlight-next-line
builder.add_conditional_edges("a", conditional_edge)

graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![条件分支图](assets/graph_api_image_5.png)

```python
result = graph.invoke({"aggregate": []})
print(result)
```
```
Adding "A" to []
Adding "C" to ['A']
{'aggregate': ['A', 'C'], 'which': 'c'}
```

!!! tip
    您的条件边可以路由到多个目标节点。例如：

    ```python
    def route_bc_or_cd(state: State) -> Sequence[str]:
        if state["which"] == "cd":
            return ["c", "d"]
        return ["b", "c"]
    ```

## Map-Reduce 和 Send API

LangGraph 使用 Send API 支持 map-reduce 和其他高级分支模式。以下是如何使用它的示例：

```python
from langgraph.graph import StateGraph, START, END, Send
from typing_extensions import TypedDict

class OverallState(TypedDict):
    topic: str
    subjects: list[str]
    jokes: list[str]
    best_selected_joke: str

def generate_topics(state: OverallState):
    return {"subjects": ["lions", "elephants", "penguins"]}

def generate_joke(state: OverallState):
    joke_map = {
        "lions": "Why don't lions like fast food? Because they can't catch it!",
        "elephants": "Why don't elephants use computers? They're afraid of the mouse!",
        "penguins": "Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice."
    }
    return {"jokes": [joke_map[state["subject"]]]}

def continue_to_jokes(state: OverallState):
    return [Send("generate_joke", {"subject": s}) for s in state["subjects"]]

def best_joke(state: OverallState):
    return {"best_selected_joke": "penguins"}

builder = StateGraph(OverallState)
builder.add_node("generate_topics", generate_topics)
builder.add_node("generate_joke", generate_joke)
builder.add_node("best_joke", best_joke)
builder.add_edge(START, "generate_topics")
builder.add_conditional_edges("generate_topics", continue_to_jokes, ["generate_joke"])
builder.add_edge("generate_joke", "best_joke")
builder.add_edge("best_joke", END)
builder.add_edge("generate_topics", END)
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![扇出 Map-Reduce 图](assets/graph_api_image_2.png)

```python
# 调用图：这里我们调用它来生成一个笑话列表
for step in graph.stream({"topic": "animals"}):
    print(step)
```
```
{'generate_topics': {'subjects': ['lions', 'elephants', 'penguins']}}
{'generate_joke': {'jokes': ["Why don't lions like fast food? Because they can't catch it!"]}}
{'generate_joke': {'jokes': ["Why don't elephants use computers? They're afraid of the mouse!"]}}
{'generate_joke': {'jokes': ['Why don't penguins like talking to strangers at parties? Because they find it hard to break the ice.']}}
{'best_joke': {'best_selected_joke': 'penguins'}}
```

## 创建和控制循环

在用循环创建图时，我们需要一种终止执行的机制。这通常是通过添加一个[条件边](../concepts/low_level.md#conditional-edges)来完成的，该边在达到某个终止条件后路由到[END](../concepts/low_level.md#end-node)节点。

您也可以在调用或流式传输图时设置图的递归限制。递归限制设置图允许执行的[超级步骤](../concepts/low_level.md#graphs)数量，然后它会引发错误。在此处阅读有关递归限制概念的更多信息。[here](../concepts/low_level.md#recursion-limit)。

让我们考虑一个带有循环的简单图，以便更好地理解这些机制的工作原理。

!!! tip
    要返回状态的 last 值而不是收到递归限制错误，请参阅[下一节](#impose-a-recursion-limit)。

创建循环时，您可以包含一个指定终止条件的条件边：

```python
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

def route(state: State) -> Literal["b", END]:
    if termination_condition(state):
        return END
    else:
        return "b"

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

要控制递归限制，请在配置中指定“recursion_limit”。这将引发 `GraphRecursionError`，您可以捕获并处理它：

```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke(inputs, {"recursion_limit": 3})
except GraphRecursionError:
    print("Recursion Error")
```

让我们定义一个带有简单循环的图。请注意，我们使用条件边来实现终止条件。

```python
import operator
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END

class State(TypedDict):
    # operator.add 缩减器函数使此为仅追加
    aggregate: Annotated[list, operator.add]

def a(state: State):
    print(f'Node A sees {state["aggregate"]}')
    return {"aggregate": ["A"]}

def b(state: State):
    print(f'Node B sees {state["aggregate"]}')
    return {"aggregate": ["B"]}

# 定义节点
builder = StateGraph(State)
builder.add_node(a)
builder.add_node(b)

# 定义边
def route(state: State) -> Literal["b", END]:
    if len(state["aggregate"]) < 7:
        return "b"
    else:
        return END

builder.add_edge(START, "a")
builder.add_conditional_edges("a", route)
builder.add_edge("b", "a")
graph = builder.compile()
```

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![简单循环图](assets/graph_api_image_3.png)

这种架构类似于[ReAct 代理](../agents/overview.md)，其中节点“a”是一个工具调用模型，节点“b”代表工具。

在我们的 `route` 条件边中，我们指定一旦状态中的“aggregate”列表长度超过某个阈值，我们就停止。

调用图，我们看到我们在终止条件达到之前在节点“a”和“b”之间交替。

```python
graph.invoke({"aggregate": []})
```
```
Node A sees []
Node B sees ['A']
Node A sees ['A', 'B']
Node B sees ['A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B']
Node B sees ['A', 'B', 'A', 'B', 'A']
Node A sees ['A', 'B', 'A', 'B', 'A', 'B']
```

### 施加递归限制

在某些应用程序中，我们可能不保证会达到给定的终止条件。在这些情况下，我们可以设置图的[递归限制](../concepts/low_level.md#recursion-limit)。这将触发 `GraphRecursionError`，经过给定数量的[超级步骤](../concepts/low_level.md#graphs)时。然后我们可以捕获并处理此异常：

```python
from langgraph.errors import GraphRecursionError

try:
    graph.invoke({"aggregate": []}, {"recursion_limit": 4})
except GraphRecursionError:
    print("Recursion Error")
```
```
Node A sees []
Node B sees ['A']
Node C sees ['A', 'B']
Node D sees ['A', 'B']
Node A sees ['A', 'B', 'C', 'D']
Recursion Error
```

??? example "扩展示例：在达到递归限制时返回状态"

    与其引发 `GraphRecursionError`，我们可以将一个新键添加到状态中，该状态跟踪剩余的步数直到达到递归限制。然后，我们可以使用此键来确定是否应结束运行。

    LangGraph 实现了一个特殊的 `RemainingSteps` 注释。在底层，它创建了一个 `ManagedValue` 通道 — 一个在我们图运行期间存在且不再存在的状态通道。

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from langgraph.managed.is_last_step import RemainingSteps

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]
        remaining_steps: RemainingSteps

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    # 定义节点
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)

    # 定义边
    def route(state: State) -> Literal["b", END]:
        if state["remaining_steps"] <= 2:
            return END
        else:
            return "b"

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "a")
    graph = builder.compile()

    # 测试一下
    result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    print(result)
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node A sees ['A', 'B']
    {'aggregate': ['A', 'B', 'A']}
    ```

??? example "扩展示例：带分支的循环"

    为了更好地理解递归限制的工作原理，让我们考虑一个更复杂的示例。下面我们实现一个循环，但一个步骤会扇出到两个节点：

    ```python
    import operator
    from typing import Annotated, Literal
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END

    class State(TypedDict):
        aggregate: Annotated[list, operator.add]

    def a(state: State):
        print(f'Node A sees {state["aggregate"]}')
        return {"aggregate": ["A"]}

    def b(state: State):
        print(f'Node B sees {state["aggregate"]}')
        return {"aggregate": ["B"]}

    def c(state: State):
        print(f'Node C sees {state["aggregate"]}')
        return {"aggregate": ["C"]}

    def d(state: State):
        print(f'Node D sees {state["aggregate"]}')
        return {"aggregate": ["D"]}

    # 定义节点
    builder = StateGraph(State)
    builder.add_node(a)
    builder.add_node(b)
    builder.add_node(c)
    builder.add_node(d)

    # 定义边
    def route(state: State) -> Literal["b", END]:
        if len(state["aggregate"]) < 7:
            return "b"
        else:
            return END

    builder.add_edge(START, "a")
    builder.add_conditional_edges("a", route)
    builder.add_edge("b", "c")
    builder.add_edge("b", "d")
    builder.add_edge(["c", "d"], "a")
    graph = builder.compile()
    ```

    ```python
    from IPython.display import Image, display

    display(Image(graph.get_graph().draw_mermaid_png()))
    ```

    ![带分支的复杂循环图](assets/graph_api_image_4.png)

    这个图看起来很复杂，但可以概念化为[超级步骤](../concepts/low_level.md#graphs)的循环：

    1. 节点 A
    2. 节点 B
    3. 节点 C 和 D
    4. 节点 A
    5. ...

    我们有一个由四个超级步骤组成的循环，其中节点 C 和 D 并行执行。

    像以前一样调用图，我们看到我们在达到终止条件之前完成了两个完整的“圈数”：

    ```python
    result = graph.invoke({"aggregate": []})
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node D sees ['A', 'B']
    Node C sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Node B sees ['A', 'B', 'C', 'D', 'A']
    Node D sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node C sees ['A', 'B', 'C', 'D', 'A', 'B']
    Node A sees ['A', 'B', 'C', 'D', 'A', 'B', 'C', 'D']
    ```

    但是，如果我们为递归限制设置为四，我们只完成一个圈，因为每个圈是四个超级步骤：

    ```python
    from langgraph.errors import GraphRecursionError

    try:
        result = graph.invoke({"aggregate": []}, {"recursion_limit": 4})
    except GraphRecursionError:
        print("Recursion Error")
    ```
    ```
    Node A sees []
    Node B sees ['A']
    Node C sees ['A', 'B']
    Node D sees ['A', 'B']
    Node A sees ['A', 'B', 'C', 'D']
    Recursion Error
    ```

## Async

使用[异步](https://docs.python.org/3/library/asyncio.html)编程范例可以为并发运行[IO 密集型](../en.wikipedia.org/wiki/I/O_bound)代码（例如，并发 API 请求到聊天模型提供商）带来显著的性能改进。

要将 `sync` 图实现转换为 `async` 实现，您需要：

1. 更新 `nodes` 以使用 `async def` 而不是 `def`。
2. 在节点内部更新代码以正确使用 `await`。
3. 根据需要使用 `.ainvoke` 或 `.astream` 调用图。

由于许多 LangChain 对象实现了[Runnable 协议](https://python.langchain.com/docs/expression_language/interface/)，它具有所有 `sync` 方法的 `async` 变体，因此通常可以快速将 `sync` 图升级为 `async` 图。

请参阅下面的示例。为了演示 LLM 的异步调用，我们将包含一个聊天模型：

{% include-markdown "../../snippets/chat_model_tabs.md" %}

```python
from langchain.chat_models import init_chat_model
from langgraph.graph import MessagesState, StateGraph

# highlight-next-line
async def node(state: MessagesState): # (1)!
    # highlight-next-line
    new_message = await llm.ainvoke(state["messages"]) # (2)!
    return {"messages": [new_message]}

builder = StateGraph(MessagesState).add_node(node).set_entry_point("node")
graph = builder.compile()

input_message = {"role": "user", "content": "Hello"}
# highlight-next-line
result = await graph.ainvoke({"messages": [input_message]}) # (3)!
```

1. 将节点声明为异步函数。
2. 在节点中使用可用时使用异步调用。
3. 使用图对象本身上的异步调用。

!!! tip "异步流式传输"
    有关使用异步进行流式传输的示例，请参阅[流式传输指南](./streaming.md)。

## 使用 `Command` 组合控制流和状态更新

将控制流（边）和状态更新（节点）结合起来可能很有用。例如，您可能希望在同一个节点中同时执行状态更新**和**决定接下来转到哪个节点。LangGraph 提供了一种通过从节点函数返回[Command](../reference/types.md#langgraph.types.Command) 对象来实现此目的的方法：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        # 状态更新
        update={"foo": "bar"},
        # 控制流
        goto="my_other_node"
    )
```

我们将在下面展示一个端到端示例。让我们创建一个包含 3 个节点（A、B 和 C）的简单图。我们首先执行节点 A，然后根据节点 A 的输出决定接下来转到节点 B 还是节点 C。

```python
import random
from typing_extensions import TypedDict, Literal
from langgraph.graph import StateGraph, START
from langgraph.types import Command

# 定义图状态
class State(TypedDict):
    foo: str

# 定义节点

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    print("Called A")
    value = random.choice(["a", "b"])
    # 这是条件边函数的替代品
    if value == "a":
        goto = "node_b"
    else:
        goto = "node_c"

    # 注意 Command 如何允许您同时更新图状态和路由到下一个节点
    return Command(
        # 这是状态更新
        update={"foo": value},
        # 这是边的替代品
        goto=goto,
    )

def node_b(state: State):
    print("Called B")
    return {"foo": state["foo"] + "b"}

def node_c(state: State):
    print("Called C")
    return {"foo": state["foo"] + "c"}
```

我们现在可以创建具有上述节点的 `StateGraph`。请注意，graph 没有[条件边](../concepts/low_level.md#conditional-edges)用于路由！这是因为控制流是在 `node_a` 内部使用 `Command` 定义的。

```python
builder = StateGraph(State)
builder.add_edge(START, "node_a")
builder.add_node(node_a)
builder.add_node(node_b)
builder.add_node(node_c)
# 注意：节点 A、B 和 C 之间没有边！

graph = builder.compile()
```

!!! important
    您可能已经注意到我们使用了 `Command` 作为返回类型注释，例如 `Command[Literal["node_b", "node_c"]]`。这对于图渲染是必需的，并告诉 LangGraph `node_a` 可以导航到 `node_b` 和 `node_c`。

```python
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

![基于 Command 的图导航](assets/graph_api_image_6.png)

如果我们多次运行该图，我们将看到它根据节点 A 中的随机选择采取不同的路径（A -> B 或 A -> C）。

```python
graph.invoke({"foo": ""})
```
```
Called A
Called C
```

### 导航到父图中的节点

如果您正在使用[子图](../concepts/subgraphs.md)，您可能希望从子图中的节点导航到另一个子图（即父图中的另一个节点）。为此，您可以在 `Command` 中指定 `graph=Command.PARENT`：

```python
def my_node(state: State) -> Command[Literal["my_other_node"]]:
    return Command(
        update={"foo": "bar"},
        goto="other_subgraph",  # 其中 `other_subgraph` 是父图中的一个节点
        graph=Command.PARENT
    )
```

让我们使用上面的示例来演示这一点。我们将通过将上面示例中的 `node_a` 更改为单个节点图来实现，然后将其作为子图添加到我们的父图中。

!!! important "使用 `Command.PARENT` 进行状态更新"
    当您从子图节点将更新发送到父图节点，以更新父子[状态模式](../concepts/low_level.md#schema)共享的键时，您**必须**在父图状态中为要更新的键定义[缩减器](../concepts/low_level.md#reducers)。请参阅下面的示例。

```python
import operator
from typing_extensions import Annotated

class State(TypedDict):
    # 注意：我们在此处定义一个缩减器
    # highlight-next-line
    foo: Annotated[str, operator.add]

def node_a(state: State) -> Command[Literal["node_b", "node_c"]]:
    print("Called A")
    value = random.choice(["a", "b"])
    # 这是条件边函数的替代品
    if value == "a":
        goto = "node_b"
    else:
        goto = "node_c"

    # 注意 Command 如何允许您同时更新图状态和路由到下一个节点
    return Command(
        update={"foo": value},
        goto=goto,
        # 这告诉 LangGraph 导航到父图中的 node_b 或 node_c
        # 注意：这将导航到相对于子图的最近的父图
        # highlight-next-line
        graph=Command.PARENT,
    )

subgraph = StateGraph(State).add_node(node_a).add_edge(START, "node_a").compile()

def node_b(state: State):
    print("Called B")
    # 注意：由于我们定义了缩减器，我们不需要手动追加
    # 现有 'foo' 值中的新字符。改用缩减器将自动追加这些
    # （通过 operator.add）
    # highlight-next-line
    return {"foo": "b"}

def node_c(state: State):
    print("Called C")
    # highlight-next-line
    return {"foo": "c"}

builder = StateGraph(State)
builder.add_edge(START, "subgraph")
builder.add_node("subgraph", subgraph)
builder.add_node(node_b)
builder.add_node(node_c)

graph = builder.compile()
```

```python
graph.invoke({"foo": ""})
```
```
Called A
Called C
```

### 在工具中使用

一个常见的用例是从工具内部更新图状态。例如，在客户支持应用程序中，您可能希望在对话开始时根据客户的账号或 ID 来查找客户信息。要从工具更新图状态，您可以从工具返回 `Command(update={"my_custom_key": "foo", "messages": [...]})`：

```python
@tool
def lookup_user_info(tool_call_id: Annotated[str, InjectedToolCallId], config: RunnableConfig):
    """Use this to look up user information to better assist them with their questions."""
    user_info = get_user_info(config.get("configurable", {}).get("user_id"))
    return Command(
        update={
            # 更新状态键
            "user_info": user_info,
            # 更新消息历史记录
            "messages": [ToolMessage("Successfully looked up user information", tool_call_id=tool_call_id)]
        }
    )
```

!!! important
    当从工具返回 `Command` 并与消息列表一起返回时（用于消息历史记录的状态键）您**必须**包括 `messages`。此消息列表**必须**包含一个 `ToolMessage`。这是有必要的，以便生成的消息历史记录有效（LLM 提供商要求工具调用后的 AI 消息后面必须有工具结果消息）。

如果您使用通过 `Command` 更新状态的工具，我们建议使用预构建的 [`ToolNode`](../reference/agents.md#langgraph.prebuilt.tool_node.ToolNode)，它会自动处理返回 `Command` 对象的工具，并将它们传播到图状态。如果您正在编写一个自定义节点来调用工具，您需要手动将工具返回的 `Command` 对象作为节点更新传播。

## 可视化您的图

这里我们演示了如何可视化您创建的图。

您可以可视化任何任意[图](https://langchain-ai.github.io/langgraph/reference/graphs/)，包括[StateGraph](https://langchain-ai.github.io/langgraph/reference/graphs.md#langgraph.graph.state.StateGraph)。让我们通过绘制分形图来玩得开心 :)。

```python
import random
from typing import Annotated, Literal
from typing_extensions import TypedDict
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

class MyNode:
    def __init__(self, name: str):
        self.name = name
    def __call__(self, state: State):
        return {"messages": [("assistant", f"Called node {self.name}")]}

def route(state) -> Literal["entry_node", "__end__"]:
    if len(state["messages"]) > 10:
        return "__end__"
    return "entry_node"

def add_fractal_nodes(builder, current_node, level, max_level):
    if level > max_level:
        return
    # 在此级别创建的节点数
    num_nodes = random.randint(1, 3)  # 根据需要调整随机性
    for i in range(num_nodes):
        nm = ["A", "B", "C"][i]
        node_name = f"node_{current_node}_{nm}"
        builder.add_node(node_name, MyNode(node_name))
        builder.add_edge(current_node, node_name)
        # 递归添加更多节点
        r = random.random()
        if r > 0.2 and level + 1 < max_level:
            add_fractal_nodes(builder, node_name, level + 1, max_level)
        elif r > 0.05:
            builder.add_conditional_edges(node_name, route, node_name)
        else:
            # 结束
            builder.add_edge(node_name, "__end__")

def build_fractal_graph(max_level: int):
    builder = StateGraph(State)
    entry_point = "entry_node"
    builder.add_node(entry_point, MyNode(entry_point))
    builder.add_edge(START, entry_point)
    add_fractal_nodes(builder, entry_point, 1, max_level)
    # 可选：如果需要，设置一个完成点
    builder.add_edge(entry_point, END)  # 或任何特定节点
    return builder.compile()

app = build_fractal_graph(3)
```

### Mermaid

我们也可以将图类转换为 Mermaid 语法。

```python
print(app.get_graph().draw_mermaid())
```
```
%%{init: {'flowchart': {'curve': 'linear'}}}%%
graph TD;
	__start__([<p>__start__</p>]):::first
	entry_node(entry_node)
	node_entry_node_A(node_entry_node_A)
	node_entry_node_B(node_entry_node_B)
	node_node_entry_node_B_A(node_node_entry_node_B_A)
	node_node_entry_node_B_B(node_node_entry_node_B_B)
	node_node_entry_node_B_C(node_node_entry_node_B_C)
	__end__([<p>__end__</p>]):::last
	__start__ --> entry_node;
	entry_node --> __end__;
	entry_node --> node_entry_node_A;
	entry_node --> node_entry_node_B;
	node_entry_node_B --> node_node_entry_node_B_A;
	node_entry_node_B --> node_node_entry_node_B_B;
	node_entry_node_B --> node_node_entry_node_B_C;
	node_entry_node_A -.-> entry_node;
	node_entry_node_A -.-> __end__;
	node_node_entry_node_B_A -.-> entry_node;
	node_node_entry_node_B_A -.-> __end__;
	node_node_entry_node_B_B -.-> entry_node;
	node_node_entry_node_B_B -.-> __end__;
	node_node_entry_node_B_C -.-> entry_node;
	node_node_entry_node_B_C -.-> __end__;
	classDef default fill:#f2f0ff,line-height:1.2
	classDef::first fill-opacity:0
	classDef::last fill:#bfb6fc
```

### PNG

如果首选，我们也可以将 Graph 渲染为 `.png`。这里我们可以使用三个选项：

- 使用 Mermaid.ink API（不需要额外的包）
- 使用 Mermaid + Pyppeteer（需要 `pip install pyppeteer`）
- 使用 graphviz（需要 `pip install graphviz`）

**使用 Mermaid.Ink**

默认情况下，`draw_mermaid_png()` 使用 Mermaid.Ink 的 API 来生成图表。

```python
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(app.get_graph().draw_mermaid_png()))
```

![分形图可视化](assets/graph_api_image_5.png)

**使用 Mermaid + Pyppeteer**

```python
import nest_asyncio

nest_asyncio.apply()  # Jupyter Notebook 运行异步函数需要此项

display(
    Image(
        app.get_graph().draw_mermaid_png(
            curve_style=CurveStyle.LINEAR,
            node_colors=NodeStyles(first="#ffdfba", last="#baffc9", default="#fad7de"),
            wrap_label_n_words=9,
            output_file_path=None,
            draw_method=MermaidDrawMethod.PYPPETEER,
            background_color="white",
            padding=10,
        )
    )
)
```

**使用 Graphviz**

```python
try:
    display(Image(app.get_graph().draw_png()))
except ImportError:
    print(
        "您可能需要安装 pygraphviz 的依赖项，更多信息请参见 https://github.com/pygraphviz/pygraphviz/blob/main/INSTALL.txt"
    )
```