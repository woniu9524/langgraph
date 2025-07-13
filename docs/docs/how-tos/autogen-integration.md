# 如何将 LangGraph 与 AutoGen、CrewAI 等框架集成

本指南介绍如何将 AutoGen 代理与 LangGraph 集成，以利用持久化、流式传输和内存等功能，然后将集成解决方案部署到 LangGraph Platform 以进行可扩展的生产使用。在本指南中，我们将演示如何构建一个与 AutoGen 集成的 LangGraph 聊天机器人，但您也可以将相同的方法应用于其他框架。

将 AutoGen 与 LangGraph 集成具有多项优势：

- **增强功能**：为您的 AutoGen 代理添加[持久化](../concepts/persistence.md)、[流式传输](../concepts/streaming.md)、[短期和长期记忆](../concepts/memory.md)等功能。
- **多代理系统**：构建一个[多代理系统](../concepts/multi_agent.md)，其中单个代理使用不同的框架构建。
- **生产部署**：将您的集成解决方案部署到[LangGraph Platform](../concepts/langgraph_platform.md) 以进行可扩展的生产使用。

## 前提条件

- Python 3.9+
- AutoGen：`pip install autogen`
- LangGraph：`pip install langgraph`
- OpenAI API 密钥

## 设置

设置您的环境：

```python
import getpass
import os


def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("OPENAI_API_KEY")
```

## 1. 定义 AutoGen 代理

创建一个可以执行代码的 AutoGen 代理。此示例改编自 AutoGen 的[官方教程](https://github.com/microsoft/autogen/blob/0.2/notebook/agentchat_web_info.ipynb)：

```python
import autogen
import os

config_list = [{"model": "gpt-4o", "api_key": os.environ["OPENAI_API_KEY"]}]

llm_config = {
    "timeout": 600,
    "cache_seed": 42,
    "config_list": config_list,
    "temperature": 0,
}

autogen_agent = autogen.AssistantAgent(
    name="assistant",
    llm_config=llm_config,
)

user_proxy = autogen.UserProxyAgent(
    name="user_proxy",
    human_input_mode="NEVER",
    max_consecutive_auto_reply=10,
    is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
    code_execution_config={
        "work_dir": "web",
        "use_docker": False,
    },  # 如果可用，请将 use_docker 设置为 True 以运行生成的代码。使用 Docker 比直接运行生成的代码更安全。
    llm_config=llm_config,
    system_message="如果任务已完全满意地解决，请回复 TERMINATE。否则，请回复 CONTINUE，或未解决任务的原因。",
)
```

## 2. 创建图表

现在我们将创建一个调用 AutoGen 代理的 LangGraph 聊天机器人图表。

```python
from langchain_core.messages import convert_to_openai_messages
from langgraph.graph import StateGraph, MessagesState, START
from langgraph.checkpoint.memory import MemorySaver

def call_autogen_agent(state: MessagesState):
    # 将 LangGraph 消息转换为 AutoGen 的 OpenAI 格式
    messages = convert_to_openai_messages(state["messages"])
    
    # 获取最后一条用户消息
    last_message = messages[-1]
    
    # 将先前的消息历史作为上下文传递（不包括最后一条消息）
    carryover = messages[:-1] if len(messages) > 1 else []
    
    # 使用 AutoGen 启动聊天
    response = user_proxy.initiate_chat(
        autogen_agent,
        message=last_message,
        carryover=carryover
    )
    
    # 从代理中提取最终响应
    final_content = response.chat_history[-1]["content"]
    
    # 以 LangGraph 格式返回响应
    return {"messages": {"role": "assistant", "content": final_content}}

# 使用内存创建带持久化的图表
checkpointer = MemorySaver()

# 构建图表
builder = StateGraph(MessagesState)
builder.add_node("autogen", call_autogen_agent)
builder.add_edge(START, "autogen")

# 使用 checkpointer 进行编译以实现持久化
graph = builder.compile(checkpointer=checkpointer)
```

```python
from IPython.display import display, Image

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Graph](./assets/autogen-output.png)

## 3. 在本地测试图表

在部署到 LangGraph Platform 之前，您可以在本地测试该图表：

```python
# 将 thread_id 传递给持久化代理输出以进行将来的交互
# highlight-next-line
config = {"configurable": {"thread_id": "1"}}

for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "找到斐波那契数列中介于 10 和 30 之间的数字",
            }
        ]
    },
    # highlight-next-line
    config,
):
    print(chunk)
```

**输出：**
```
user_proxy (to assistant):

找到斐波那契数列中介于 10 和 30 之间的数字

--------------------------------------------------------------------------------
assistant (to user_proxy):

要找到斐波那契数列中介于 10 和 30 之间的数字，我们可以生成斐波那契数列并检查哪些数字在此范围内。这是一个计划：

1. 从 0 开始生成斐波那契数。
2. 继续生成，直到数字超过 30。
3. 收集并打印介于 10 和 30 之间的数字。

...
```

由于我们利用了 LangGraph 的[持久化](https://langchain-ai.github.io/langgraph/concepts/persistence/)功能，现在我们可以使用相同的 thread_id 继续对话——LangGraph 将自动将先前的历史记录传递给 AutoGen 代理：

```python
for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "将最后一个数字乘以 3",
            }
        ]
    },
    # highlight-next-line
    config,
):
    print(chunk)
```

**输出：**
```
user_proxy (to assistant):

将最后一个数字乘以 3
上下文： 
找到斐波那契数列中介于 10 和 30 之间的数字
斐波那契数列中介于 10 和 30 之间的数字是 13 和 21。 

这些数字是斐波那契数列的一部分，该数列通过将前两个数字相加来得到下一个数字，从 0 和 1 开始。 

数列为：0, 1, 1, 2, 3, 5, 8, 13, 21, 34, ...

如您所见，13 和 21 是此数列中唯一介于 10 和 30 之间的数字。 

TERMINATE

--------------------------------------------------------------------------------
assistant (to user_proxy):

斐波那契数列中介于 10 和 30 之间的最后一个数字是 21。将 21 乘以 3 得到：

21 * 3 = 63

TERMINATE

--------------------------------------------------------------------------------
{'call_autogen_agent': {'messages': {'role': 'assistant', 'content': '斐波那契数列中介于 10 和 30 之间的最后一个数字是 21。将 21 乘以 3 得到：\n\n21 * 3 = 63\n\nTERMINATE'}}}
``` 

## 4. 为部署做准备

要部署到 LangGraph Platform，请创建类似以下的目录结构：

```
my-autogen-agent/
├── agent.py          # 您的主要代理代码
├── requirements.txt  # Python 依赖项
└── langgraph.json   # LangGraph 配置
```

=== "agent.py"

    ```python
    import os
    import autogen
    from langchain_core.messages import convert_to_openai_messages
    from langgraph.graph import StateGraph, MessagesState, START
    from langgraph.checkpoint.memory import MemorySaver

    # AutoGen 配置
    config_list = [{"model": "gpt-4o", "api_key": os.environ["OPENAI_API_KEY"]}]

    llm_config = {
        "timeout": 600,
        "cache_seed": 42,
        "config_list": config_list,
        "temperature": 0,
    }

    # 创建 AutoGen 代理
    autogen_agent = autogen.AssistantAgent(
        name="assistant",
        llm_config=llm_config,
    )

    user_proxy = autogen.UserProxyAgent(
        name="user_proxy",
        human_input_mode="NEVER",
        max_consecutive_auto_reply=10,
        is_termination_msg=lambda x: x.get("content", "").rstrip().endswith("TERMINATE"),
        code_execution_config={
            "work_dir": "/tmp/autogen_work",
            "use_docker": False,
        },
        llm_config=llm_config,
        system_message="如果任务已完全满意地解决，请回复 TERMINATE。",
    )

    def call_autogen_agent(state: MessagesState):
        """调用 AutoGen 代理的节点函数"""
        messages = convert_to_openai_messages(state["messages"])
        last_message = messages[-1]
        carryover = messages[:-1] if len(messages) > 1 else []
        
        response = user_proxy.initiate_chat(
            autogen_agent,
            message=last_message,
            carryover=carryover
        )
        
        final_content = response.chat_history[-1]["content"]
        return {"messages": {"role": "assistant", "content": final_content}}

    # 创建并编译图表
    def create_graph():
        checkpointer = MemorySaver()
        builder = StateGraph(MessagesState)
        builder.add_node("autogen", call_autogen_agent)
        builder.add_edge(START, "autogen")
        return builder.compile(checkpointer=checkpointer)

    # 导出图表以供 LangGraph Platform 使用
    graph = create_graph()
    ```

=== "requirements.txt"

    ```
    langgraph>=0.1.0
    pyautogen>=0.2.0
    langchain-core>=0.1.0
    langchain-openai>=0.0.5
    ```

=== "langgraph.json"

    ```json
    {
    "dependencies": ["."],
    "graphs": {
        "autogen_agent": "./agent.py:graph"
    },
    "env": ".env"
    }
    ```


## 5. 部署到 LangGraph Platform

使用 LangGraph Platform CLI 部署图表：

```
pip install -U langgraph-cli
```

```
langgraph deploy --config langgraph.json 
```