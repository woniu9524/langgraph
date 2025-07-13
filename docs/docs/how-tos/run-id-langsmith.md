# 如何为 LangSmith 中的图执行传递自定义运行 ID 或设置标签和元数据

!!! tip "先决条件"
    本指南假设您熟悉以下内容：

    - [LangSmith 文档](https://docs.smith.langchain.com)
    - [LangSmith 平台](https://smith.langchain.com)
    - [RunnableConfig](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.config.RunnableConfig.html#langchain_core.runnables.config.RunnableConfig)
    - [为跟踪添加元数据和标签](https://docs.smith.langchain.com/how_to_guides/tracing/trace_with_langchain#add-metadata-and-tags-to-traces)
    - [自定义运行名称](https://docs.smith.langchain.com/how_to_guides/tracing/trace_with_langchain#customize-run-name)

在 IDE 或终端中调试图执行有时会很困难。[LangSmith](https://docs.smith.langchain.com) 允许您使用跟踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用 — 阅读[LangSmith 文档](https://docs.smith.langchain.com)以获取有关如何开始的更多信息。

为了更容易地识别和分析在图调用期间生成的跟踪，您可以在运行时设置附加配置（请参阅[RunnableConfig](https://api.python.langchain.com/en/latest/runnables/langchain_core.runnables.config.RunnableConfig.html#langchain_core.runnables.config.RunnableConfig)）：

| **字段** | **类型** | **描述** |
|-------------|---------------------|--------------------------------------------------------------------------------------------------------------------|
| run_name | `str` | 此调用跟踪运行的名称。默认为类名。 |
| run_id | `UUID` | 此调用跟踪运行的唯一标识符。如果未提供，将生成新的 UUID。 |
| tags | `List[str]` | 此调用及任何子调用（例如，调用 LLM 的 Chain）的标签。您可以使用它们来过滤调用。 |
| metadata | `Dict[str, Any]` | 此调用及任何子调用（例如，调用 LLM 的 Chain）的元数据。键应该是字符串，值应该是 JSON 可序列化的。 |

LangGraph 图实现了[LangChain Runnable 接口](https://python.langchain.com/api_reference/core/runnables/langchain_core.runnables.base.Runnable.html)，并在 `invoke`、`ainvoke`、`stream` 等方法中接受第二个参数 (`RunnableConfig`)。

LangSmith 平台将允许您根据 `run_name`、`run_id`、`tags` 和 `metadata` 搜索和过滤跟踪。

## 简而言之

```python
import uuid
# 生成一个随机 UUID——它必须是一个 UUID
config = {"run_id": uuid.uuid4()}, "tags": ["my_tag1"], "metadata": {"a": 5}}
# 适用于所有标准的 Runnable 方法
# 如 invoke, batch, ainvoke, astream_events 等
graph.stream(inputs, config, stream_mode="values")
```

本指南的其余部分将展示一个完整的代理。

## 设置

首先，让我们安装所需的包并设置我们的 API 密钥

```python
%%capture --no-stderr
%pip install --quiet -U langgraph langchain_openai
```

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("OPENAI_API_KEY")
_set_env("LANGSMITH_API_KEY")
```

!!! tip
    注册 LangSmith 以快速发现问题并提高 LangGraph 项目的性能。[LangSmith](https://docs.smith.langchain.com) 允许您使用跟踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用 — 在[此处](https://docs.smith.langchain.com)了解有关如何开始的更多信息。

## 定义 Graph

在本例中，我们将使用[预构建的 ReAct 代理](https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/)。

```python
from langchain_openai import ChatOpenAI
from typing import Literal
from langgraph.prebuilt import create_react_agent
from langchain_core.tools import tool

# 首先我们初始化要使用的的模型。
model = ChatOpenAI(model="gpt-4o", temperature=0)


# 对于本教程，我们将使用自定义工具，该工具返回两个城市（NYC 和 SF）的预定义天气值
@tool
def get_weather(city: Literal["nyc", "sf"]):
    """使用此工具获取天气信息。"""
    if city == "nyc":
        return "It might be cloudy in nyc"
    elif city == "sf":
        return "It's always sunny in sf"
    else:
        raise AssertionError("Unknown city")


tools = [get_weather]


# 定义图
graph = create_react_agent(model, tools=tools)
```

## 运行 Graph

现在我们已经定义了我们的图，让我们运行一次并在 LangSmith 中查看跟踪。为了方便在 LangSmith 中访问我们的跟踪，我们将向配置中传递一个自定义的 `run_id`。

这假设您已设置 `LANGSMITH_API_KEY` 环境变量。

请注意，您还可以通过设置 `LANGCHAIN_PROJECT` 环境变量来配置要跟踪的项目，默认情况下，运行将跟踪到 `default` 项目。

```python
import uuid


def print_stream(stream):
    for s in stream:
        message = s["messages"][-1]
        if isinstance(message, tuple):
            print(message)
        else:
            message.pretty_print()


inputs = {"messages": [("user", "what is the weather in sf")]}

config = {"run_name": "agent_007", "tags": ["cats are awesome"]}

print_stream(graph.stream(inputs, config, stream_mode="values"))
```

**输出：**
```
================================ Human Message ==================================

what is the weather in sf
================================== Ai Message ===================================
Tool Calls:
  get_weather (call_9ZudXyMAdlUjptq9oMGtQo8o)
 Call ID: call_9ZudXyMAdlUjptq9oMGtQo8o
  Args:
    city: sf
================================= Tool Message ==================================
Name: get_weather

It's always sunny in sf
================================== Ai Message ===================================

The weather in San Francisco is currently sunny.
```

## 在 LangSmith 中查看跟踪

现在我们已经运行了我们的图，让我们前往 LangSmith 查看我们的跟踪。首先，点击您跟踪到的项目（在本例中为默认项目）。您应该会看到一个具有自定义运行名称“agent_007”的运行。

![LangSmith Trace View](assets/d38d1f2b-0f4c-4707-b531-a3c749de987f.png)

此外，您将能够事后使用提供的标签或元数据来过滤跟踪。例如，

![LangSmith Filter View](assets/410e0089-2ab8-46bb-a61a-827187fd46b3.png)