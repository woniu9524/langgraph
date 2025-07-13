# LangGraph 预构建

该库定义了用于创建和执行 LangGraph 代理和工具的高级 API。

> [!IMPORTANT]
> 此库旨在与 `langgraph` 一起打包安装，请勿直接安装它。

## 代理 (Agents)

`langgraph-prebuilt` 提供了一个用于工具调用的 [ReAct 风格](https://langchain-ai.github.io/langgraph/concepts/agentic_concepts/#react-implementation) 代理的实现 - `create_react_agent`：

```bash
pip install langchain-anthropic
```

```python
from langchain_anthropic import ChatAnthropic
from langgraph.prebuilt import create_react_agent

# 定义代理要使用的工具
def search(query: str):
    """调用以浏览网页。"""
    # 这是一个占位符，但不要告诉 LLM……
    if "sf" in query.lower() or "san francisco" in query.lower():
        return "It's 60 degrees and foggy."
    return "It's 90 degrees and sunny."

tools = [search]
model = ChatAnthropic(model="claude-3-7-sonnet-latest")

app = create_react_agent(model, tools)
# 运行代理
app.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]},
)
```

## 工具 (Tools)

### ToolNode

`langgraph-prebuilt` 提供了一个用于执行工具调用的节点的实现 - `ToolNode`：

```python
from langgraph.prebuilt import ToolNode
from langchain_core.messages import AIMessage

def search(query: str):
    """调用以浏览网页。"""
    # 这是一个占位符，但不要告诉 LLM……
    if "sf" in query.lower() or "san francisco" in query.lower():
        return "It's 60 degrees and foggy."
    return "It's 90 degrees and sunny."

tool_node = ToolNode([search])
tool_calls = [{"name": "search", "args": {"query": "what is the weather in sf"}, "id": "1"}]
ai_message = AIMessage(content="", tool_calls=tool_calls)
# 执行工具调用
tool_node.invoke({"messages": [ai_message]})
```

### ValidationNode

`langgraph-prebuilt` 提供了一个用于将工具调用与 pydantic 模式一起验证的节点的实现 - `ValidationNode`：

```python
from pydantic import BaseModel, field_validator
from langgraph.prebuilt import ValidationNode
from langchain_core.messages import AIMessage


class SelectNumber(BaseModel):
    a: int

    @field_validator("a")
    def a_must_be_meaningful(cls, v):
        if v != 37:
            raise ValueError("Only 37 is allowed")
        return v

validation_node = ValidationNode([SelectNumber])
validation_node.invoke({
    "messages": [AIMessage("", tool_calls=[{"name": "SelectNumber", "args": {"a": 42}, "id": "1"}])]
})
```

## Agent Inbox

该库包含用于将 [Agent Inbox](https://github.com/langchain-ai/agent-inbox) 与 LangGraph 代理一起使用的模式。在此处了解有关如何使用 Agent Inbox 的更多信息 ([https://github.com/langchain-ai/agent-inbox#interrupts](https://github.com/langchain-ai/agent-inbox#interrupts))。

```python
from langgraph.types import interrupt
from langgraph.prebuilt.interrupt import HumanInterrupt, HumanResponse

def my_graph_function():
    # 从状态的 `messages` 字段中提取最后一个工具调用
    tool_call = state["messages"][-1].tool_calls[0]
    # 创建一个中断
    request: HumanInterrupt = {
        "action_request": {
            "action": tool_call['name'],
            "args": tool_call['args']
        },
        "config": {
            "allow_ignore": True,
            "allow_respond": True,
            "allow_edit": False,
            "allow_accept": False
        },
        "description": _generate_email_markdown(state) # 生成详细的 markdown 描述。
    }
    # 将中断请求作为列表发送，并提取第一个响应
    response = interrupt([request])[0]
    if response['type'] == "response":
        # 做一些与响应相关的事情
    ...
```