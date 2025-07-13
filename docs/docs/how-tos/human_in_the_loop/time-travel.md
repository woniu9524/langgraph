# 使用时间旅行

要 LangGraph 中使用 [时间旅行](../../concepts/time-travel.md):

1. 使用 [`invoke`][langgraph.graph.state.CompiledStateGraph.invoke] 或 [`stream`][langgraph.graph.state.CompiledStateGraph.stream] 方法，并提供初始输入来[运行图表](#1-run-the-graph)。
2. [识别现有线程中的检查点](#2-identify-a-checkpoint): 使用 [`get_state_history()`][langgraph.graph.state.CompiledStateGraph.get_state_history] 方法检索特定 `thread_id` 的执行历史，并定位所需的 `checkpoint_id`。 或者，在您希望执行暂停的节点（或节点之前）设置一个[中断](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。然后，您可以找到直到该中断为止记录的最新检查点。
3. [更新图状态（可选）](#3-update-the-state-optional): 使用 [`update_state`][langgraph.graph.state.CompiledStateGraph.update_state] 方法修改检查点处的图状态，并从备用状态恢复执行。
4. [从检查点恢复执行](#4-resume-execution-from-the-checkpoint): 使用 `invoke` 或 `stream` 方法，输入为 `None`，并提供包含相应 `thread_id` 和 `checkpoint_id` 的配置。

!!! tip

    有关时间旅行的概念性概述，请参阅[时间旅行](../../concepts/time-travel.md)。

## 在工作流中

此示例构建了一个简单的 LangGraph 工作流，该工作流生成一个笑话主题并使用 LLM 编写一个笑话。它演示了如何运行图表、检索过去的执行检查点、可选地修改状态以及从选定的检查点恢复执行以探索备用结果。

### 设置

首先，我们需要安装所需的包

```python
%%capture --no-stderr
%pip install --quiet -U langgraph langchain_anthropic
```

接下来，我们需要设置 Anthropic（我们将使用的 LLM）的 API 密钥

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")
```

<div class="admonition tip">
    <p class="admonition-title">为 LangGraph 开发设置 <a href="https://smith.langchain.com">LangSmith</a></p>
    <p style="padding-top: 5px;">
        注册 LangSmith 以快速发现问题并提高 LangGraph 项目的性能。LangSmith 允许您使用跟踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用——在此处<a href="https://docs.smith.langchain.com">了解更多关于如何开始的信息</a>。
    </p>
</div>

```python
import uuid

from typing_extensions import TypedDict, NotRequired
from langgraph.graph import StateGraph, START, END
from langchain.chat_models import init_chat_model
from langgraph.checkpoint.memory import InMemorySaver


class State(TypedDict):
    topic: NotRequired[str]
    joke: NotRequired[str]


llm = init_chat_model(
    "anthropic:claude-3-7-sonnet-latest",
    temperature=0,
)


def generate_topic(state: State):
    """LLM 调用以生成笑话主题"""
    msg = llm.invoke("Give me a funny topic for a joke")
    return {"topic": msg.content}


def write_joke(state: State):
    """LLM 调用以根据主题写笑话"""
    msg = llm.invoke(f"Write a short joke about {state['topic']}")
    return {"joke": msg.content}


# 构建工作流
workflow = StateGraph(State)

# 添加节点
workflow.add_node("generate_topic", generate_topic)
workflow.add_node("write_joke", write_joke)

# 添加边以连接节点
workflow.add_edge(START, "generate_topic")
workflow.add_edge("generate_topic", "write_joke")
workflow.add_edge("write_joke", END)

# 编译
checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)
graph
```

### 1. 运行图表

```python
config = {
    "configurable": {
        "thread_id": uuid.uuid4(),
    }
}
state = graph.invoke({}, config)

print(state["topic"])
print()
print(state["joke"])
```

**输出:**
```
How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don't know about? There's a lot of comedic potential in the everyday mystery that unites us all!

# The Secret Life of Socks in the Dryer

I finally discovered where all my missing socks go after the dryer. Turns out they're not missing at all—they've just eloped with someone else's socks from the laundromat to start new lives together.

My blue argyle is now living in Bermuda with a red polka dot, posting vacation photos on Sockstagram and sending me lint as alimony.
```

### 2. 识别检查点

```python
# 状态按时间倒序返回。
states = list(graph.get_state_history(config))

for state in states:
    print(state.next)
    print(state.config["configurable"]["checkpoint_id"])
    print()
```

**输出:**
```
()
1f02ac4a-ec9f-6524-8002-8f7b0bbeed0e

('write_joke',)
1f02ac4a-ce2a-6494-8001-cb2e2d651227

('generate_topic',)
1f02ac4a-a4e0-630d-8000-b73c254ba748

('__start__',)
1f02ac4a-a4dd-665e-bfff-e6c8c44315d9
```

```python
# 这是倒数第二个状态（状态按时间顺序排列）
selected_state = states[1]
print(selected_state.next)
print(selected_state.values)
```

**输出:**
```
('write_joke',)
{'topic': 'How about "The Secret Life of Socks in the Dryer"? You know, exploring the mysterious phenomenon of how socks go into the laundry as pairs but come out as singles. Where do they go? Are they starting new lives elsewhere? Is there a sock paradise we don\\'t know about? There\\'s a lot of comedic potential in the everyday mystery that unites us all!'}
```

### 3. 更新状态（可选）

`update_state` 将创建一个新的检查点。新检查点将与同一线程关联，但具有新的检查点 ID。

```python
new_config = graph.update_state(selected_state.config, values={"topic": "chickens"})
print(new_config)
```

**输出:**
```
{'configurable': {'thread_id': 'c62e2e03-c27b-4cb6-8cea-ea9bfedae006', 'checkpoint_ns': '', 'checkpoint_id': '1f02ac4a-ecee-600b-8002-a1d21df32e4c'}}
```

### 4. 从检查点恢复执行

```python
graph.invoke(None, new_config)
```

**输出:**
```python
{'topic': 'chickens',
 'joke': 'Why did the chicken join a band?\n\nBecause it had excellent drumsticks!'}
```