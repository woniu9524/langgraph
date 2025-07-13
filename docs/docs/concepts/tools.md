# 工具

许多 AI 应用通过自然语言与用户进行交互。然而，一些用例要求模型使用结构化输入直接与外部系统接口，例如 API、数据库或文件系统。在这些场景中，[工具调用](../how-tos/tool-calling.md) 使模型能够生成符合指定输入架构的请求。

**工具**封装了一个可调用的函数及其输入架构。这些工具可以传递给兼容的 [聊天模型](https://python.langchain.com/docs/concepts/chat_models)，允许模型决定是否以及使用什么参数来调用工具。

## 工具调用

![模型工具调用图示](./img/tool_call.png)

工具调用通常是**条件性的**。基于用户输入和可用工具，模型可以选择发出工具调用请求。此请求将在 `AIMessage` 对象中返回，其中包含一个 `tool_calls` 字段，指定工具名称和输入参数：

```python
llm_with_tools.invoke("What is 2 multiplied by 3?")
# -> AIMessage(tool_calls=[{'name': 'multiply', 'args': {'a': 2, 'b': 3}, ...}])
```

如果输入与任何工具都无关，模型将仅返回自然语言消息：

```python
llm_with_tools.invoke("Hello world!")  # -> AIMessage(content="Hello!")
```

重要的是，模型本身不执行工具，它只生成一个请求。一个单独的执行器（例如运行时或代理）负责处理工具调用并返回结果。

有关更多详细信息，请参阅 [工具调用指南](../how-tos/tool-calling.md)。

## 预置工具

LangChain 提供了针对常见外部系统的预置工具集成，包括 API、数据库、文件系统和 Web 数据。

浏览 [集成目录](https://python.langchain.com/docs/integrations/tools/) 以查找可用工具。

常见类别：

*   **搜索**：Bing、SerpAPI、Tavily
*   **代码执行**：Python REPL、Node.js REPL
*   **数据库**：SQL、MongoDB、Redis
*   **Web 数据**：抓取和浏览
*   **API**：OpenWeatherMap、NewsAPI 等。

## 自定义工具

您可以使用 `@tool` 装饰器或普通 Python 函数定义自定义工具。例如：

```python
from langchain_core.tools import tool

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

有关更多详细信息，请参阅 [工具调用指南](../how-tos/tool-calling.md)。

## 工具执行

虽然模型决定何时调用工具，但工具调用的执行必须由运行时组件处理。

LangGraph 提供了这方面的预置组件：

*   [`ToolNode`][langgraph.prebuilt.tool_node.ToolNode]：一个预置的执行工具的节点。
*   [`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent]：构建一个完全自主管理工具调用的代理。