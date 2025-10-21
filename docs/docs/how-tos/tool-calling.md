# 调用工具

[工具（Tools）](../concepts/tools.md)封装了可调用的函数及其输入模式（schema）。这些工具可以传递给兼容的聊天模型，让模型决定是否调用一个工具以及确定合适的参数。

你可以[定义自己的工具](#define-a-tool)，或者使用[预构建的工具](#prebuilt-tools)。

## 定义工具

:::python
使用 `@tool` 装饰器定义一个基础工具：

```python
from langchain_core.tools import tool

# highlight-next-line
@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

:::

:::js
使用 `tool` 函数定义一个基础工具：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const multiply = tool(
  (input) => {
    return input.a * input.b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers.",
    schema: z.object({
      a: z.number().describe("First operand"),
      b: z.number().describe("Second operand"),
    }),
  }
);
```

:::

## 运行工具

工具符合[Runnable 接口](https://python.langchain.com/docs/concepts/runnables/)，这意味着你可以使用 `invoke` 方法来运行一个工具：

:::python

```python
multiply.invoke({"a": 6, "b": 7})  # 返回 42
```

:::

:::js

```typescript
await multiply.invoke({ a: 6, b: 7 }); // 返回 42
```

:::

如果工具使用 `type="tool_call"` 调用，它将返回一个 [ToolMessage](https://python.langchain.com/docs/concepts/messages/#toolmessage)：

:::python

```python
tool_call = {
    "type": "tool_call",
    "id": "1",
    "args": {"a": 42, "b": 7}
}
multiply.invoke(tool_call) # 返回一个 ToolMessage 对象
```

输出：

```pycon
ToolMessage(content='294', name='multiply', tool_call_id='1')
```

:::

:::js

```typescript
const toolCall = {
  type: "tool_call",
  id: "1",
  name: "multiply",
  args: { a: 42, b: 7 },
};
await multiply.invoke(toolCall); // 返回一个 ToolMessage 对象
```

输出：

```
ToolMessage {
  content: "294",
  name: "multiply",
  tool_call_id: "1"
}
```

:::

## 在 Agent 中使用

:::python
要创建可以调用工具的 Agent，你可以使用预构建的 @[create_react_agent][create_react_agent]：

```python
from langchain_core.tools import tool
# highlight-next-line
from langgraph.prebuilt import create_react_agent

@tool
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b

# highlight-next-line
agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet",
    tools=[multiply]
)
agent.invoke({"messages": [{"role": "user", "content": "what's 42 x 7?"}]})
```

:::

:::js
要创建可以调用工具的 Agent，你可以使用预构建的 [createReactAgent](https://js.langchain.com/docs/api/langgraph_prebuilt/functions/createReactAgent.html)：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
// highlight-next-line
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const multiply = tool(
  (input) => {
    return input.a * input.b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers.",
    schema: z.object({
      a: z.number().describe("First operand"),
      b: z.number().describe("Second operand"),
    }),
  }
);

// highlight-next-line
const agent = createReactAgent({
  llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
  tools: [multiply],
});

await agent.invoke({
  messages: [{ role: "user", content: "what's 42 x 7?" }],
});
```

:::

:::python

### 动态选择工具

根据上下文在运行时配置工具可用性：

```python
from dataclasses import dataclass
from typing import Literal

from langchain.chat_models import init_chat_model
from langchain_core.tools import tool

from langgraph.prebuilt import create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState
from langgraph.runtime import Runtime


@dataclass
class CustomContext:
    tools: list[Literal["weather", "compass"]]


@tool
def weather() -> str:
    """Returns the current weather conditions."""
    return "It's nice and sunny."


@tool
def compass() -> str:
    """Returns the direction the user is facing."""
    return "North"

model = init_chat_model("anthropic:claude-sonnet-4-20250514")

# highlight-next-line
def configure_model(state: AgentState, runtime: Runtime[CustomContext]):
    """Configure the model with tools based on runtime context."""
    selected_tools = [
        tool
        for tool in [weather, compass]
        if tool.name in runtime.context.tools
    ]
    return model.bind_tools(selected_tools)


agent = create_react_agent(
    # Dynamically configure the model with tools based on runtime context
    # highlight-next-line
    configure_model,
    # Initialize with all tools available
    # highlight-next-line
    tools=[weather, compass]
)

output = agent.invoke(
    {"messages": [{"role": "user", "content": "Who are you and what tools do you have access to?", }]},
    # highlight-next-line
    context=CustomContext(tools=["weather"]),  # Only enable the weather tool
)

print(output["messages"][-1].text())
```

!!! version-added "Added in version 0.6.0"

:::

## 在工作流中使用

如果你正在编写自定义工作流，你需要：

1.  将工具注册到聊天模型。
2.  如果模型决定使用该工具，则调用该工具。

:::python
使用 `model.bind_tools()` 将工具注册到模型。

```python
from langchain.chat_models import init_chat_model

model = init_chat_model(model="claude-3-5-haiku-latest")

# highlight-next-line
model_with_tools = model.bind_tools([multiply])
```

:::

:::js
使用 `model.bindTools()` 将工具注册到模型。

```typescript
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({ model: "gpt-4o" });

// highlight-next-line
const modelWithTools = model.bindTools([multiply]);
```

:::

LLM 会自动确定是否需要调用工具，并处理使用适当参数调用该工具。

??? example "扩展示例：将工具附加到聊天模型"

    :::python
    ```python
    from langchain_core.tools import tool
    from langchain.chat_models import init_chat_model

    @tool
    def multiply(a: int, b: int) -> int:
        """Multiply two numbers."""
        return a * b

    model = init_chat_model(model="claude-3-5-haiku-latest")
    # highlight-next-line
    model_with_tools = model.bind_tools([multiply])

    response_message = model_with_tools.invoke("what's 42 x 7?")
    tool_call = response_message.tool_calls[0]

    multiply.invoke(tool_call)
    ```

    ```pycon
    ToolMessage(
        content='294',
        name='multiply',
        tool_call_id='toolu_0176DV4YKSD8FndkeuuLj36c'
    )
    ```
    :::

    :::js
    ```typescript
    import { tool } from "@langchain/core/tools";
    import { ChatOpenAI } from "@langchain/openai";
    import { z } from "zod";

    const multiply = tool(
      (input) => {
        return input.a * input.b;
      },
      {
        name: "multiply",
        description: "Multiply two numbers.",
        schema: z.object({
          a: z.number().describe("First operand"),
          b: z.number().describe("Second operand"),
        }),
      }
    );

    const model = new ChatOpenAI({ model: "gpt-4o" });
    // highlight-next-line
    const modelWithTools = model.bindTools([multiply]);

    const responseMessage = await modelWithTools.invoke("what's 42 x 7?");
    const toolCall = responseMessage.tool_calls[0];

    await multiply.invoke(toolCall);
    ```

    ```
    ToolMessage {
      content: "294",
      name: "multiply",
      tool_call_id: "toolu_0176DV4YKSD8FndkeuuLj36c"
    }
    ```
    :::

#### ToolNode

:::python
要在自定义工作流中执行工具，请使用预构建的 @[`ToolNode`][ToolNode] 或实现自己的自定义节点。

`ToolNode` 是一个专门用于在工作流中执行工具的节点，它提供以下功能：

- 支持同步和异步工具。
- 并发执行多个工具。
- 处理工具执行期间的错误（`handle_tool_errors=True`，默认启用）。有关更多详细信息，请参阅[处理工具错误](#handle-errors)。

`ToolNode` 在 [`MessagesState`](../concepts/low_level.md#messagesstate) 上运行：

- **输入**：`MessagesState`，最后一个消息是包含 `tool_calls` 参数的 `AIMessage`。
- **输出**：`MessagesState`，已通过执行工具产生的 [`ToolMessage`](https://python.langchain.com/docs/concepts/messages/#toolmessage) 更新。

```python
# highlight-next-line
from langgraph.prebuilt import ToolNode

def get_weather(location: str):
    """Call to get the current weather."""
    if location.lower() in ["sf", "san francisco"]:
        return "It's 60 degrees and foggy."
    else:
        return "It's 90 degrees and sunny."

def get_coolest_cities():
    """Get a list of coolest cities"""
    return "nyc, sf"

# highlight-next-line
tool_node = ToolNode([get_weather, get_coolest_cities])
tool_node.invoke({"messages": [...]})
```

:::

:::js
要在自定义工作流中执行工具，请使用预构建的 [`ToolNode`](https://js.langchain.com/docs/api/langgraph_prebuilt/classes/ToolNode.html) 或实现自己的自定义节点。

`ToolNode` 是一个专门用于在工作流中执行工具的节点，它提供以下功能：

- 支持同步和异步工具。
- 并发执行多个工具。
- 处理工具执行期间的错误（`handleToolErrors: true`，默认启用）。有关更多详细信息，请参阅[处理工具错误](#handle-errors)。

- **输入**：`MessagesZodState`，最后一个消息是包含 `tool_calls` 参数的 `AIMessage`。
- **输出**：`MessagesZodState`，已通过执行工具产生的 [`ToolMessage`](https://js.langchain.com/docs/concepts/messages/#toolmessage) 更新。

```typescript
// highlight-next-line
import { ToolNode } from "@langchain/langgraph/prebuilt";

const getWeather = tool(
  (input) => {
    if (["sf", "san francisco"].includes(input.location.toLowerCase())) {
      return "It's 60 degrees and foggy.";
    } else {
      return "It's 90 degrees and sunny.";
    }
  },
  {
    name: "get_weather",
    description: "Call to get the current weather.",
    schema: z.object({
      location: z.string().describe("Location to get the weather for."),
    }),
  }
);

const getCoolestCities = tool(
  () => {
    return "nyc, sf";
  },
  {
    name: "get_coolest_cities",
    description: "Get a list of coolest cities",
    schema: z.object({
      noOp: z.string().optional().describe("No-op parameter."),
    }),
  }
);

// highlight-next-line
const toolNode = new ToolNode([getWeather, getCoolestCities]);
await toolNode.invoke({ messages: [...] });
```

:::

??? example "单个工具调用"

    :::python
    ```python
    from langchain_core.messages import AIMessage
    from langgraph.prebuilt import ToolNode

    # Define tools
    @tool
    def get_weather(location: str):
        """Call to get the current weather."""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."

    # highlight-next-line
    tool_node = ToolNode([get_weather])

    message_with_single_tool_call = AIMessage(
        content="",
        tool_calls=[
            {
                "name": "get_weather",
                "args": {"location": "sf"},
                "id": "tool_call_id",
                "type": "tool_call",
            }
        ],
    )

    tool_node.invoke({"messages": [message_with_single_tool_call]})
    ```

    ```
    {'messages': [ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id')]}
    ```
    :::

    :::js
    ```typescript
    import { AIMessage } from "@langchain/core/messages";
    import { ToolNode } from "@langchain/langgraph/prebuilt";
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";

    // Define tools
    const getWeather = tool(
      (input) => {
        if (["sf", "san francisco"].includes(input.location.toLowerCase())) {
          return "It's 60 degrees and foggy.";
        } else {
          return "It's 90 degrees and sunny.";
        }
      },
      {
        name: "get_weather",
        description: "Call to get the current weather.",
        schema: z.object({
          location: z.string().describe("Location to get the weather for."),
        }),
      }
    );

    // highlight-next-line
    const toolNode = new ToolNode([getWeather]);

    const messageWithSingleToolCall = new AIMessage({
      content: "",
      tool_calls: [
        {
          name: "get_weather",
          args: { location: "sf" },
          id: "tool_call_id",
          type: "tool_call",
        }
      ],
    });

    await toolNode.invoke({ messages: [messageWithSingleToolCall] });
    ```

    ```
    { messages: [ToolMessage { content: "It's 60 degrees and foggy.", name: "get_weather", tool_call_id: "tool_call_id" }] }
    ```
    :::

??? example "多个工具调用"

    :::python
    ```python
    from langchain_core.messages import AIMessage
    from langgraph.prebuilt import ToolNode

    # Define tools

    def get_weather(location: str):
        """Call to get the current weather."""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."

    def get_coolest_cities():
        """Get a list of coolest cities"""
        return "nyc, sf"

    # highlight-next-line
    tool_node = ToolNode([get_weather, get_coolest_cities])

    message_with_multiple_tool_calls = AIMessage(
        content="",
        tool_calls=[
            {
                "name": "get_coolest_cities",
                "args": {},
                "id": "tool_call_id_1",
                "type": "tool_call",
            },
            {
                "name": "get_weather",
                "args": {"location": "sf"},
                "id": "tool_call_id_2",
                "type": "tool_call",
            },
        ],
    )

    # highlight-next-line
    tool_node.invoke({"messages": [message_with_multiple_tool_calls]})  # (1)!
    ```

    1. `ToolNode` 将并行执行两个工具。

    ```
    {
        'messages': [
            ToolMessage(content='nyc, sf', name='get_coolest_cities', tool_call_id='tool_call_id_1'),
            ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='tool_call_id_2')
        ]
    }
    ```
    :::

    :::js
    ```typescript
    import { AIMessage } from "@langchain/core/messages";
    import { ToolNode } from "@langchain/langgraph/prebuilt";
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";

    // Define tools
    const getWeather = tool(
      (input) => {
        if (["sf", "san francisco"].includes(input.location.toLowerCase())) {
          return "It's 60 degrees and foggy.";
        } else {
          return "It's 90 degrees and sunny.";
        }
      },
      {
        name: "get_weather",
        description: "Call to get the current weather.",
        schema: z.object({
          location: z.string().describe("Location to get the weather for."),
        }),
      }
    );

    const getCoolestCities = tool(
      () => {
        return "nyc, sf";
      },
      {
        name: "get_coolest_cities",
        description: "Get a list of coolest cities",
        schema: z.object({
          noOp: z.string().optional().describe("No-op parameter."),
        }),
      }
    );

    // highlight-next-line
    const toolNode = new ToolNode([getWeather, getCoolestCities]);

    const messageWithMultipleToolCalls = new AIMessage({
      content: "",
      tool_calls: [
        {
          name: "get_coolest_cities",
          args: {},
          id: "tool_call_id_1",
          type: "tool_call",
        },
        {
          name: "get_weather",
          args: { location: "sf" },
          id: "tool_call_id_2",
          type: "tool_call",
        },
      ],
    });

    // highlight-next-line
    await toolNode.invoke({ messages: [messageWithMultipleToolCalls] }); // (1)!
    ```

    1. `ToolNode` 将并行执行两个工具。

    ```
    {
      messages: [
        ToolMessage { content: "nyc, sf", name: "get_coolest_cities", tool_call_id: "tool_call_id_1" },
        ToolMessage { content: "It's 60 degrees and foggy.", name: "get_weather", tool_call_id: "tool_call_id_2" }
      ]
    }
    ```
    :::

??? example "与聊天模型一起使用"

    :::python
    ```python
    from langchain.chat_models import init_chat_model
    from langgraph.prebuilt import ToolNode

    def get_weather(location: str):
        """Call to get the current weather."""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."

    # highlight-next-line
    tool_node = ToolNode([get_weather])

    model = init_chat_model(model="claude-3-5-haiku-latest")
    # highlight-next-line
    model_with_tools = model.bind_tools([get_weather])  # (1)!


    # highlight-next-line
    response_message = model_with_tools.invoke("what's the weather in sf?")
    tool_node.invoke({"messages": [response_message]})
    ```

    1. 使用 `.bind_tools()` 将工具模式附加到聊天模型。

    ```
    {'messages': [ToolMessage(content="It's 60 degrees and foggy.", name='get_weather', tool_call_id='toolu_01Pnkgw5JeTRxXAU7tyHT4UW')]}
    ```
    :::

    :::js
    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { ToolNode } from "@langchain/langgraph/prebuilt";
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";

    const getWeather = tool(
      (input) => {
        if (["sf", "san francisco"].includes(input.location.toLowerCase())) {
          return "It's 60 degrees and foggy.";
        } else {
          return "It's 90 degrees and sunny.";
        }
      },
      {
        name: "get_weather",
        description: "Call to get the current weather.",
        schema: z.object({
          location: z.string().describe("Location to get the weather for."),
        }),
      }
    );

    // highlight-next-line
    const toolNode = new ToolNode([getWeather]);

    const model = new ChatOpenAI({ model: "gpt-4o" });
    // highlight-next-line
    const modelWithTools = model.bindTools([getWeather]); // (1)!

    // highlight-next-line
    const responseMessage = await modelWithTools.invoke("what's the weather in sf?");
    await toolNode.invoke({ messages: [responseMessage] });
    ```

    1. 使用 `.bindTools()` 将工具模式附加到聊天模型。

    ```
    { messages: [ToolMessage { content: "It's 60 degrees and foggy.", name: "get_weather", tool_call_id: "toolu_01Pnkgw5JeTRxXAU7tyHT4UW" }] }
    ```
    :::

??? example "在工具调用 Agent 中使用"

    这是从头开始使用 `ToolNode` 创建一个工具调用 Agent 的示例。你也可以使用 LangGraph 的预构建 [Agent](../agents/agents.md)。

    :::python
    ```python
    from langchain.chat_models import init_chat_model
    from langgraph.prebuilt import ToolNode
    from langgraph.graph import StateGraph, MessagesState, START, END

    def get_weather(location: str):
        """Call to get the current weather."""
        if location.lower() in ["sf", "san francisco"]:
            return "It's 60 degrees and foggy."
        else:
            return "It's 90 degrees and sunny."

    # highlight-next-line
    tool_node = ToolNode([get_weather])

    model = init_chat_model(model="claude-3-5-haiku-latest")
    # highlight-next-line
    model_with_tools = model.bind_tools([get_weather])

    def should_continue(state: MessagesState):
        messages = state["messages"]
        last_message = messages[-1]
        if last_message.tool_calls:
            return "tools"
        return END

    def call_model(state: MessagesState):
        messages = state["messages"]
        response = model_with_tools.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState)

    # Define the two nodes we will cycle between
    builder.add_node("call_model", call_model)
    # highlight-next-line
    builder.add_node("tools", tool_node)

    builder.add_edge(START, "call_model")
    builder.add_conditional_edges("call_model", should_continue, ["tools", END])
    builder.add_edge("tools", "call_model")

    graph = builder.compile()

    graph.invoke({"messages": [{"role": "user", "content": "what's the weather in sf?"}]})
    ```

    ```
    {
        'messages': [
            HumanMessage(content="what's the weather in sf?"),
            AIMessage(
                content=[{'text': "I'll help you check the weather in San Francisco right now.", 'type': 'text'}, {'id': 'toolu_01A4vwUEgBKxfFVc5H3v1CNs', 'input': {'location': 'San Francisco'}, 'name': 'get_weather', 'type': 'tool_use'}],
                tool_calls=[{'name': 'get_weather', 'args': {'location': 'San Francisco'}, 'id': 'toolu_01A4vwUEgBKxfFVc5H3v1CNs', 'type': 'tool_call'}]
            ),
            ToolMessage(content="It's 60 degrees and foggy."),
            AIMessage(content="The current weather in San Francisco is 60 degrees and foggy. Typical San Francisco weather with its famous marine layer!")
        ]
    }
    ```
    :::

    :::js
    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { ToolNode } from "@langchain/langgraph/prebuilt";
    import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { isAIMessage } from "@langchain/core/messages";

    const getWeather = tool(
      (input) => {
        if (["sf", "san francisco"].includes(input.location.toLowerCase())) {
          return "It's 60 degrees and foggy.";
        } else {
          return "It's 90 degrees and sunny.";
        }
      },
      {
        name: "get_weather",
        description: "Call to get the current weather.",
        schema: z.object({
          location: z.string().describe("Location to get the weather for."),
        }),
      }
    );

    // highlight-next-line
    const toolNode = new ToolNode([getWeather]);

    const model = new ChatOpenAI({ model: "gpt-4o" });
    // highlight-next-line
    const modelWithTools = model.bindTools([getWeather]);

    const shouldContinue = (state: z.infer<typeof MessagesZodState>) => {
      const messages = state.messages;
      const lastMessage = messages.at(-1);
      if (lastMessage && isAIMessage(lastMessage) && lastMessage.tool_calls?.length) {
        return "tools";
      }
      return END;
    };

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = state.messages;
      const response = await modelWithTools.invoke(messages);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      // Define the two nodes we will cycle between
      .addNode("agent", callModel)
      // highlight-next-line
      .addNode("tools", toolNode)
      .addEdge(START, "agent")
      .addConditionalEdges("agent", shouldContinue, ["tools", END])
      .addEdge("tools", "agent");

    const graph = builder.compile();

    await graph.invoke({
      messages: [{ role: "user", content: "what's the weather in sf?" }]
    });
    ```

    ```
    {
      messages: [
        HumanMessage { content: "what's the weather in sf?" },
        AIMessage {
          content: [{ text: "I'll help you check the weather in San Francisco right now.", type: "text" }, { id: "toolu_01A4vwUEgBKxfFVc5H3v1CNs", input: { location: "San Francisco" }, name: "get_weather", type: "tool_use" }],
          tool_calls: [{ name: "get_weather", args: { location: "San Francisco" }, id: "toolu_01A4vwUEgBKxfFVc5H3v1CNs", type: "tool_call" }]
        },
        ToolMessage { content: "It's 60 degrees and foggy." },
        AIMessage { content: "The current weather in San Francisco is 60 degrees and foggy. Typical San Francisco weather with its famous marine layer!" }
      ]
    }
    ```
    :::

## 工具自定义

要对工具行为进行更精细地控制，请使用 `@tool` 装饰器。

### 参数描述

:::python
从文档字符串中自动生成描述：

```python
# highlight-next-line
from langchain_core.tools import tool

# highlight-next-line
@tool("multiply_tool", parse_docstring=True)
def multiply(a: int, b: int) -> int:
    """Multiply two numbers.

    Args:
        a: First operand
        b: Second operand
    """
    return a * b
```

:::

:::js
从模式（schema）中自动生成描述：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const multiply = tool(
  (input) => {
    return input.a * input.b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers.",
    schema: z.object({
      a: z.number().describe("First operand"),
      b: z.number().describe("Second operand"),
    }),
  }
);
```

:::

### 显式输入模式

:::python
使用 `args_schema` 定义模式：

```python
from pydantic import BaseModel, Field
from langchain_core.tools import tool

class MultiplyInputSchema(BaseModel):
    """Multiply two numbers"""
    a: int = Field(description="First operand")
    b: int = Field(description="Second operand")

# highlight-next-line
@tool("multiply_tool", args_schema=MultiplyInputSchema)
def multiply(a: int, b: int) -> int:
    return a * b
```

:::

### 工具名称

使用第一个参数或 name 属性来覆盖默认工具名称：

:::python

```python
from langchain_core.tools import tool

# highlight-next-line
@tool("multiply_tool")
def multiply(a: int, b: int) -> int:
    """Multiply two numbers."""
    return a * b
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const multiply = tool(
  (input) => {
    return input.a * input.b;
  },
  {
    name: "multiply_tool", // Custom name
    description: "Multiply two numbers.",
    schema: z.object({
      a: z.number().describe("First operand"),
      b: z.number().describe("Second operand"),
    }),
  }
);
```

:::

## 上下文管理

LangGraph 中的工具有时需要上下文数据，例如不应由模型控制的运行时参数（如用户 ID 或会话详细信息）。LangGraph 提供了三种管理此类上下文的方法：

| 类型                     | 使用场景                           | 可变性 | 生命周期           |
| ------------------------ | ---------------------------------- | ------ | ------------------ |
| [配置（Configuration）](#configuration) | 静态、不可变的运行时数据           | ❌     | 单次调用           |
| [短期记忆（Short-term memory）](#short-term-memory) | 调用过程中的动态、变化的数据           | ✅     | 单次调用           |
| [长期记忆（Long-term memory）](#long-term-memory)   | 持久化的、跨会话的数据           | ✅     | 跨多个会话         |

### 配置

:::python
当你有 **不可变** 的运行时数据供工具使用时（例如用户标识符），请使用配置。你可以在调用时通过 [`RunnableConfig`](https://python.langchain.com/docs/concepts/runnables/#runnableconfig) 传递这些参数，并在工具中访问它们：

```python
from langchain_core.tools import tool
from langchain_core.runnables import RunnableConfig

@tool
# highlight-next-line
def get_user_info(config: RunnableConfig) -> str:
    """Retrieve user information based on user ID."""
    user_id = config["configurable"].get("user_id")
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

# 带有 Agent 的调用示例
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user info"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

:::

:::js
当你有 **不可变** 的运行时数据供工具使用时（例如用户标识符），请使用配置。你可以在调用时通过 [`LangGraphRunnableConfig`](https://js.langchain.com/docs/api/langgraph/interfaces/LangGraphRunnableConfig.html) 传递这些参数，并在工具中访问它们：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import type { LangGraphRunnableConfig } from "@langchain/langgraph";

const getUserInfo = tool(
  // highlight-next-line
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config?.configurable?.user_id;
    return userId === "user_123" ? "User is John Smith" : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Retrieve user information based on user ID.",
    schema: z.object({}),
  }
);

// 带有 Agent 的调用示例
await agent.invoke(
  { messages: [{ role: "user", content: "look up user info" }] },
  // highlight-next-line
  { configurable: { user_id: "user_123" } }
);
```

:::

??? example "扩展示例：在工具中访问配置"

    :::python
    ```python
    from langchain_core.runnables import RunnableConfig
    from langchain_core.tools import tool
    from langgraph.prebuilt import create_react_agent

    def get_user_info(
        # highlight-next-line
        config: RunnableConfig,
    ) -> str:
        """Look up user info."""
        # highlight-next-line
        user_id = config["configurable"].get("user_id")
        return "User is John Smith" if user_id == "user_123" else "Unknown user"

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_user_info],
        state_schema=CustomState,
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "look up user information"}]},
        # highlight-next-line
        config={"configurable": {"user_id": "user_123"}}
    )
    ```
    :::

    :::js
    ```typescript
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { ChatAnthropic } from "@langchain/anthropic";
    import type { LangGraphRunnableConfig } from "@langchain/langgraph";

    const getUserInfo = tool(
      // highlight-next-line
      async (_, config: LangGraphRunnableConfig) => {
        // highlight-next-line
        const userId = config?.configurable?.user_id;
        return userId === "user_123" ? "User is John Smith" : "Unknown user";
      },
      {
        name: "get_user_info",
        description: "Look up user info.",
        schema: z.object({}),
      }
    );

    const agent = createReactAgent({
      llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
      tools: [getUserInfo],
    });

    await agent.invoke(
      { messages: [{ role: "user", content: "look up user information" }] },
      // highlight-next-line
      { configurable: { user_id: "user_123" } }
    );
    ```
    :::

### 短期记忆

短期记忆在单次执行过程中维护**动态**状态，该状态会在其中发生变化。

:::python
要**访问**（读取）工具中的图状态，可以使用特殊的参数**注解** — @[`InjectedState`][InjectedState]：

```python
from typing import Annotated, NotRequired
from langchain_core.tools import tool
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState

class CustomState(AgentState):
    # 用户名字段在短期状态中
    user_name: NotRequired[str]

@tool
def get_user_name(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """Retrieve the current user-name from state."""
    # Return stored name or a default if not set
    return state.get("user_name", "Unknown user")

# 示例 Agent 设置
agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_name],
    state_schema=CustomState,
)

# 调用：从状态读取名称（最初为空）
agent.invoke({"messages": "what's my name?"})
```

:::

:::js
要**访问**（读取）工具中的图状态，可以使用 `getContextVariable` 函数：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { getContextVariable } from "@langchain/core/context";
import { MessagesZodState } from "@langchain/langgraph";
import type { LangGraphRunnableConfig } from "@langchain/langgraph";

const getUserName = tool(
  // highlight-next-line
  async (_, config: LangGraphRunnableConfig) => {
    // highlight-next-line
    const currentState = getContextVariable("currentState") as z.infer<
      typeof MessagesZodState
    > & { userName?: string };
    return currentState?.userName || "Unknown user";
  },
  {
    name: "get_user_name",
    description: "Retrieve the current user name from state.",
    schema: z.object({}),
  }
);
```

:::

:::python
使用返回 `Command` 的工具来**更新** `user_name` 并附加一个确认消息：

```python
from typing import Annotated
from langgraph.types import Command
from langchain_core.messages import ToolMessage
from langchain_core.tools import tool, InjectedToolCallId

@tool
def update_user_name(
    new_name: str,
    tool_call_id: Annotated[str, InjectedToolCallId]
) -> Command:
    """Update user-name in short-term memory."""
    # highlight-next-line
    return Command(update={
        # highlight-next-line
        "user_name": new_name,
        # highlight-next-line
        "messages": [
            # highlight-next-line
            ToolMessage(f"Updated user name to {new_name}", tool_call_id=tool_call_id)
            # highlight-next-line
        ]
        # highlight-next-line
    })
```

:::

:::js
若要**更新**短期记忆，你可以使用返回 `Command` 以更新状态的工具：

```typescript
import { Command } from "@langchain/langgraph";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const updateUserName = tool(
  async (input) => {
    // highlight-next-line
    return new Command({
      // highlight-next-line
      update: {
        // highlight-next-line
        userName: input.newName,
        // highlight-next-line
        messages: [
          // highlight-next-line
          {
            // highlight-next-line
            role: "assistant",
            // highlight-next-line
            content: `Updated user name to ${input.newName}`,
            // highlight-next-line
          },
          // highlight-next-line
        ],
        // highlight-next-line
      },
      // highlight-next-line
    });
  },
  {
    name: "update_user_name",
    description: "Update user name in short-term memory.",
    schema: z.object({
      newName: z.string().describe("The new user name"),
    }),
  }
);
```

:::

!!! important

    :::python
    如果要使用返回 `Command` 并更新图状态的工具，你可以使用预构建的 @[`create_react_agent`][create_react_agent] / @[`ToolNode`][ToolNode] 组件，或者实现自己的工具执行节点，该节点收集工具返回的 `Command` 对象，并返回一个列表，例如：

    ```python
    def call_tools(state):
        ...
        commands = [tools_by_name[tool_call["name"]].invoke(tool_call) for tool_call in tool_calls]
        return commands
    ```
    :::

    :::js
    如果要使用返回 `Command` 并更新图状态的工具，你可以使用预构建的 @[`createReactAgent`][create_react_agent] / @[ToolNode] 组件，或者实现自己的工具执行节点，该节点收集工具返回的 `Command` 对象，并返回一个列表，例如：

    ```typescript
    const callTools = async (state: State) => {
      // ...
      const commands = await Promise.all(
        toolCalls.map(toolCall => toolsByName[toolCall.name].invoke(toolCall))
      );
      return commands;
    };
    ```
    :::

### 长期记忆

使用[长期记忆](../concepts/memory.md#long-term-memory)跨对话存储用户特定或应用程序特定的数据。这对于聊天机器人等应用程序非常有用，您可以在其中记住用户偏好或其他信息。

要使用长期记忆，您需要：

1. [配置存储](memory/add-memory.md#add-long-term-memory)以在调用之间持久化数据。
2. 从工具内部访问存储。

:::python
要**访问**存储中的信息：

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.tools import tool
from langgraph.graph import StateGraph
# highlight-next-line
from langgraph.config import get_store

@tool
def get_user_info(config: RunnableConfig) -> str:
    """Look up user info."""
    # 与提供给 `builder.compile(store=store)`
    # 或 `create_react_agent` 的内容相同
    # highlight-next-line
    store = get_store()
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    user_info = store.get(("users",), user_id)
    return str(user_info.value) if user_info else "Unknown user"

builder = StateGraph(...)
...
graph = builder.compile(store=store)
```

:::

:::js
要**访问**存储中的信息：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import type { LangGraphRunnableConfig } from "@langchain/langgraph";

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    // 与提供给 `builder.compile({ store })`
    // 或 `createReactAgent` 的内容相同
    // highlight-next-line
    const store = config.store;
    if (!store) throw new Error("Store not provided");

    const userId = config?.configurable?.user_id;
    // highlight-next-line
    const userInfo = await store.get(["users"], userId);
    return userInfo?.value ? JSON.stringify(userInfo.value) : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);
```

:::

??? example "访问长期记忆"

    :::python
    ```python
    from langchain_core.runnables import RunnableConfig
    from langchain_core.tools import tool
    from langgraph.config import get_store
    from langgraph.prebuilt import create_react_agent
    from langgraph.store.memory import InMemoryStore

    # highlight-next-line
    store = InMemoryStore() # (1)!

    # highlight-next-line
    store.put(  # (2)!
        ("users",),  # (3)!
        "user_123",  # (4)!
        {
            "name": "John Smith",
            "language": "English",
        } # (5)!
    )

    @tool
    def get_user_info(config: RunnableConfig) -> str:
        """Look up user info."""
        # 与提供给 `create_react_agent` 的内容相同
        # highlight-next-line
        store = get_store() # (6)!
        user_id = config["configurable"].get("user_id")
        # highlight-next-line
        user_info = store.get(("users",), user_id) # (7)!
        return str(user_info.value) if user_info else "Unknown user"

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[get_user_info],
        # highlight-next-line
        store=store # (8)!
    )

    # Run the agent
    agent.invoke(
        {"messages": [{"role": "user", "content": "look up user information"}]},
        # highlight-next-line
        config={"configurable": {"user_id": "user_123"}}
    )
    ```

    1. `InMemoryStore` 是一个在内存中存储数据的存储。在生产环境中，通常会使用数据库或其他持久化存储。请查阅[存储文档](../reference/store.md)，了解更多选项。如果你使用**LangGraph Platform**进行部署，该平台将为你提供生产级的存储。
    2. 在此示例中，我们使用 `put` 方法向存储中写入一些示例数据。有关更多详细信息，请参阅 @[BaseStore.put] API 参考。
    3. 第一个参数是名称空间。该参数用于将相关数据分组在一起。在此示例中，我们使用 `users` 名称空间来分组用户数据。
    4. 名称空间内的键。此示例使用用户 ID 作为键。
    5. 我们要为给定用户存储的数据。
    6. `get_store` 函数用于访问存储。你可以从代码的任何位置调用它，包括工具和提示。此函数返回创建 Agent 时传递给 Agent 的存储。
    7. `get` 方法用于从存储中检索数据。第一个参数是名称空间，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关该值元数据。
    8. `store` 被传递给 Agent。这使得 Agent 在运行工具时可以访问存储。你也可以使用 `get_store` 函数从代码的任何位置访问存储。
    :::

    :::js
    ```typescript
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { InMemoryStore } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/anthropic";
    import type { LangGraphRunnableConfig } from "@langchain/langgraph";

    // highlight-next-line
    const store = new InMemoryStore(); // (1)!

    // highlight-next-line
    await store.put(  // (2)!
      ["users"],  // (3)!
      "user_123",  // (4)!
      {
        name: "John Smith",
        language: "English",
      } // (5)!
    );

    const getUserInfo = tool(
      async (_, config: LangGraphRunnableConfig) => {
        // 与提供给 `createReactAgent` 的内容相同
        // highlight-next-line
        const store = config.store; // (6)!
        if (!store) throw new Error("Store not provided");

        const userId = config?.configurable?.user_id;
        // highlight-next-line
        const userInfo = await store.get(["users"], userId); // (7)!
        return userInfo?.value ? JSON.stringify(userInfo.value) : "Unknown user";
      },
      {
        name: "get_user_info",
        description: "Look up user info.",
        schema: z.object({}),
      }
    );

    const agent = createReactAgent({
      llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
      tools: [getUserInfo],
      // highlight-next-line
      store: store // (8)!
    });

    // Run the agent
    await agent.invoke(
      { messages: [{ role: "user", content: "look up user information" }] },
      // highlight-next-line
      { configurable: { user_id: "user_123" } }
    );
    ```

    1. `InMemoryStore` 是一个在内存中存储数据的存储。在生产环境中，通常会使用数据库或其他持久化存储。请查阅[存储文档](../reference/store.md)，了解更多选项。如果你使用**LangGraph Platform**进行部署，该平台将为你提供生产级的存储。
    2. 在此示例中，我们使用 `put` 方法向存储中写入一些示例数据。有关更多详细信息，请参阅 [BaseStore.put](https://js.langchain.com/docs/api/langgraph_store/classes/BaseStore.html#put) API 参考。
    3. 第一个参数是名称空间。该参数用于将相关数据分组在一起。在此示例中，我们使用 `users` 名称空间来分组用户数据。
    4. 名称空间内的键。此示例使用用户 ID 作为键。
    5. 我们要为给定用户存储的数据。
    6. 存储可以通过传递给工具的配置对象进行访问。这使得工具在运行时可以访问存储。
    7. `get` 方法用于从存储中检索数据。第一个参数是名称空间，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关该值元数据。
    8. `store` 被传递给 Agent。这使得 Agent 在运行工具时可以访问存储。
    :::

:::python
要**更新**存储中的信息：

```python
from langchain_core.runnables import RunnableConfig
from langchain_core.tools import tool
from langgraph.graph import StateGraph
# highlight-next-line
from langgraph.config import get_store

@tool
def save_user_info(user_info: str, config: RunnableConfig) -> str:
    """Save user info."""
    # 与提供给 `builder.compile(store=store)`
    # 或 `create_react_agent` 的内容相同
    # highlight-next-line
    store = get_store()
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    store.put(("users",), user_id, user_info)
    return "Successfully saved user info."

builder = StateGraph(...)
...
graph = builder.compile(store=store)
```

:::

:::js
要**更新**存储中的信息：

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import type { LangGraphRunnableConfig } from "@langchain/langgraph";

const saveUserInfo = tool(
  async (input, config: LangGraphRunnableConfig) => {
    // 与提供给 `builder.compile({ store })`
    // 或 `createReactAgent` 的内容相同
    // highlight-next-line
    const store = config.store;
    if (!store) throw new Error("Store not provided");

    const userId = config?.configurable?.user_id;
    // highlight-next-line
    await store.put(["users"], userId, input.userInfo);
    return "Successfully saved user info.";
  },
  {
    name: "save_user_info",
    description: "Save user info.",
    schema: z.object({
      userInfo: z.string().describe("User information to save"),
    }),
  }
);
```

:::

??? example "更新长期记忆"

    :::python
    ```python
    from typing_extensions import TypedDict

    from langchain_core.tools import tool
    from langgraph.config import get_store
    from langchain_core.runnables import RunnableConfig
    from langgraph.prebuilt import create_react_agent
    from langgraph.store.memory import InMemoryStore

    store = InMemoryStore() # (1)!

    class UserInfo(TypedDict): # (2)!
        name: str

    @tool
    def save_user_info(user_info: UserInfo, config: RunnableConfig) -> str: # (3)!
        """Save user info."""
        # 与提供给 `create_react_agent` 的内容相同
        # highlight-next-line
        store = get_store() # (4)!
        user_id = config["configurable"].get("user_id")
        # highlight-next-line
        store.put(("users",), user_id, user_info) # (5)!
        return "Successfully saved user info."

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[save_user_info],
        # highlight-next-line
        store=store
    )

    # Run the agent
    agent.invoke(
        {"messages": [{"role": "user", "content": "My name is John Smith"}]},
        # highlight-next-line
        config={"configurable": {"user_id": "user_123"}} # (6)!
    )

    # You can access the store directly to get the value
    store.get(("users",), "user_123").value
    ```

    1. `InMemoryStore` 是一个在内存中存储数据的存储。在生产环境中，通常会使用数据库或其他持久化存储。请查阅[存储文档](../reference/store.md)，了解更多选项。如果你使用**LangGraph Platform**进行部署，该平台将为你提供生产级的存储。
    2. `UserInfo` 类是一个 `TypedDict`，它定义了用户信息结构的结构。LLM 将使用此结构根据模式格式化响应。
    3. `save_user_info` 函数是一个工具，允许 Agent 更新用户信息。这对于聊天应用程序很有用，用户可以在其中更新其个人资料信息。
    4. `get_store` 函数用于访问存储。你可以从代码的任何位置调用它，包括工具和提示。此函数返回创建 Agent 时传递给 Agent 的存储。
    5. `put` 方法用于将数据存储在存储中。第一个参数是名称空间，第二个参数是键。这将把用户信息存储在存储中。
    6. `user_id` 在配置中传入。这用于标识正在更新信息的用户的 ID。
    :::

    :::js
    ```typescript
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { InMemoryStore } from "@langchain/langgraph";
    import { ChatAnthropic } from "@langchain/anthropic";
    import type { LangGraphRunnableConfig } from "@langchain/langgraph";

    const store = new InMemoryStore(); // (1)!

    const UserInfoSchema = z.object({ // (2)!
      name: z.string(),
    });

    const saveUserInfo = tool(
      async (input, config: LangGraphRunnableConfig) => { // (3)!
        // 与提供给 `createReactAgent` 的内容相同
        // highlight-next-line
        const store = config.store; // (4)!
        if (!store) throw new Error("Store not provided");

        const userId = config?.configurable?.user_id;
        // highlight-next-line
        await store.put(["users"], userId, input); // (5)!
        return "Successfully saved user info.";
      },
      {
        name: "save_user_info",
        description: "Save user info.",
        schema: UserInfoSchema,
      }
    );

    const agent = createReactAgent({
      llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
      tools: [saveUserInfo],
      // highlight-next-line
      store: store
    });

    // Run the agent
    await agent.invoke(
      { messages: [{ role: "user", content: "My name is John Smith" }] },
      // highlight-next-line
      { configurable: { user_id: "user_123" } } // (6)!
    );

    // You can access the store directly to get the value
    const userInfo = await store.get(["users"], "user_123");
    console.log(userInfo?.value);
    ```

    1. `InMemoryStore` 是一个在内存中存储数据的存储。在生产环境中，通常会使用数据库或其他持久化存储。请查阅[存储文档](../reference/store.md)，了解更多选项。如果你使用**LangGraph Platform**进行部署，该平台将为你提供生产级的存储。
    2. `UserInfoSchema` 是一个 Zod 模式，它定义了用户信息结构的结构。LLM 将使用此结构根据模式格式化响应。
    3. `saveUserInfo` 函数是一个工具，允许 Agent 更新用户信息。这对于聊天应用程序很有用，用户可以在其中更新其个人资料信息。
    4. 存储可以通过传递给工具的配置对象进行访问。这使得工具在运行时可以访问存储。
    5. `put` 方法用于将数据存储在存储中。第一个参数是名称空间，第二个参数是键。这将把用户信息存储在存储中。
    6. `user_id` 在配置中传入。这用于标识正在更新信息的用户的 ID。
    :::

## 高级工具功能

### 即时返回

:::python
使用 `return_direct=True` 可在执行其他逻辑之前立即返回工具的结果。

这对于不应触发进一步处理或工具调用的工具很有用，允许你直接将结果返回给用户。

```python
# highlight-next-line
@tool(return_direct=True)
def add(a: int, b: int) -> int:
    """Add two numbers"""
    return a + b
```

:::

:::js
使用 `returnDirect: true` 可在执行其他逻辑之前立即返回工具的结果。

这对于不应触发进一步处理或工具调用的工具很有用，允许你直接将结果返回给用户。

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const add = tool(
  (input) => {
    return input.a + input.b;
  },
  {
    name: "add",
    description: "Add two numbers",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
    // highlight-next-line
    returnDirect: true,
  }
);
```

:::

??? example "扩展示例：在预构建 Agent 中使用 return_direct"

    :::python
    ```python
    from langchain_core.tools import tool
    from langgraph.prebuilt import create_react_agent

    # highlight-next-line
    @tool(return_direct=True)
    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    agent = create_react_agent(
        model="anthropic:claude-3-7-sonnet-latest",
        tools=[add]
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what's 3 + 5?"}]}
    )
    ```
    :::

    :::js
    ```typescript
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { ChatAnthropic } from "@langchain/anthropic";

    // highlight-next-line
    const add = tool(
      (input) => {
        return input.a + input.b;
      },
      {
        name: "add",
        description: "Add two numbers",
        schema: z.object({
          a: z.number(),
          b: z.number(),
        }),
        // highlight-next-line
        returnDirect: true,
      }
    );

    const agent = createReactAgent({
      llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
      tools: [add]
    });

    await agent.invoke({
      messages: [{ role: "user", content: "what's 3 + 5?" }]
    });
    ```
    :::

!!! important "未使用预构建组件时"

    :::python
    如果你正在构建自定义工作流，并且不依赖 `create_react_agent` 或 `ToolNode`，你还需要
    实现控制流来处理 `return_direct=True`。
    :::

    :::js
    如果你正在构建自定义工作流，并且不依赖 `createReactAgent` 或 `ToolNode`，你还需要
    实现控制流来处理 `returnDirect: true`。
    :::

### 强制使用工具

如果你需要强制使用特定工具，你需要在**模型**级别进行配置，使用 `bind_tools` 方法中的 `tool_choice` 参数。

通过 `tool_choice` 强制特定工具使用：

:::python

```python
@tool(return_direct=True)
def greet(user_name: str) -> int:
    """Greet user."""
    return f"Hello {user_name}!"

tools = [greet]

configured_model = model.bind_tools(
    tools,
    # Force the use of the 'greet' tool
    # highlight-next-line
    tool_choice={"type": "tool", "name": "greet"}
)
```

:::

:::js

```typescript
const greet = tool(
  (input) => {
    return `Hello ${input.userName}!`;
  },
  {
    name: "greet",
    description: "Greet user.",
    schema: z.object({
      userName: z.string(),
    }),
    returnDirect: true,
  }
);

const tools = [greet];

const configuredModel = model.bindTools(
  tools,
  // Force the use of the 'greet' tool
  // highlight-next-line
  { tool_choice: { type: "tool", name: "greet" } }
);
```

:::

??? example "扩展示例：在 Agent 中强制使用工具"

    :::python
    要在 Agent 中强制使用特定工具，可以在 `model.bind_tools()` 中设置 `tool_choice` 选项：

    ```python
    from langchain_core.tools import tool

    # highlight-next-line
    @tool(return_direct=True)
    def greet(user_name: str) -> int:
        """Greet user."""
        return f"Hello {user_name}!"

    tools = [greet]

    agent = create_react_agent(
        # highlight-next-line
        model=model.bind_tools(tools, tool_choice={"type": "tool", "name": "greet"}),
        tools=tools
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "Hi, I am Bob"}]}
    )
    ```
    :::

    :::js
    要在 Agent 中强制使用特定工具，可以在 `model.bindTools()` 中设置 `tool_choice` 选项：

    ```typescript
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { ChatOpenAI } from "@langchain/openai";

    // highlight-next-line
    const greet = tool(
      (input) => {
        return `Hello ${input.userName}!`;
      },
      {
        name: "greet",
        description: "Greet user.",
        schema: z.object({
          userName: z.string(),
        }),
        // highlight-next-line
        returnDirect: true,
      }
    );

    const tools = [greet];
    const model = new ChatOpenAI({ model: "gpt-4o" });

    const agent = createReactAgent({
      // highlight-next-line
      llm: model.bindTools(tools, { tool_choice: { type: "tool", name: "greet" } }),
      tools: tools
    });

    await agent.invoke({
      messages: [{ role: "user", content: "Hi, I am Bob" }]
    });
    ```
    :::

!!! Warning "避免无限循环"

    :::python
    强制使用工具而不设置停止条件可能会导致无限循环。使用以下任一保护措施：

    - 将工具标记为 [`return_direct=True`](#immediate-return)，以在执行后结束循环。
    - 设置 [`recursion_limit`](../concepts/low_level.md#recursion-limit) 来限制执行步骤数。
    :::

    :::js
    强制使用工具而不设置停止条件可能会导致无限循环。使用以下任一保护措施：

    - 将工具标记为 [`returnDirect: true`](#immediate-return)，以在执行后结束循环。
    - 设置 [`recursionLimit`](../concepts/low_level.md#recursion-limit) 来限制执行步骤数。
    :::

!!! tip "工具选择配置"

    `tool_choice` 参数用于配置模型在决定调用工具时应使用哪个工具。如果你想确保始终为特定任务调用某个工具，或者想覆盖模型基于其内部逻辑选择工具的默认行为，这将非常有用。

    请注意，并非所有模型都支持此功能，具体配置可能因你使用的模型而异。

### 禁用并行调用

:::python
对于受支持的提供商，可以通过 `model.bind_tools()` 方法设置 `parallel_tool_calls=False` 来禁用并行工具调用：

```python
model.bind_tools(
    tools,
    # highlight-next-line
    parallel_tool_calls=False
)
```

:::

:::js
对于受支持的提供商，可以通过 `model.bindTools()` 方法设置 `parallel_tool_calls: false` 来禁用并行工具调用：

```typescript
model.bindTools(
  tools,
  // highlight-next-line
  { parallel_tool_calls: false }
);
```

:::

??? example "扩展示例：在预构建 Agent 中禁用并行工具调用"

    :::python
    ```python
    from langchain.chat_models import init_chat_model

    def add(a: int, b: int) -> int:
        """Add two numbers"""
        return a + b

    def multiply(a: int, b: int) -> int:
        """Multiply two numbers."""
        return a * b

    model = init_chat_model("anthropic:claude-3-5-sonnet-latest", temperature=0)
    tools = [add, multiply]
    agent = create_react_agent(
        # disable parallel tool calls
        # highlight-next-line
        model=model.bind_tools(tools, parallel_tool_calls=False),
        tools=tools
    )

    agent.invoke(
        {"messages": [{"role": "user", "content": "what's 3 + 5 and 4 * 7?"}]}
    )
    ```
    :::

    :::js
    ```typescript
    import { ChatOpenAI } from "@langchain/openai";
    import { tool } from "@langchain/core/tools";
    import { z } from "zod";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    const add = tool(
      (input) => {
        return input.a + input.b;
      },
      {
        name: "add",
        description: "Add two numbers",
        schema: z.object({
          a: z.number(),
          b: z.number(),
        }),
      }
    );

    const multiply = tool(
      (input) => {
        return input.a * input.b;
      },
      {
        name: "multiply",
        description: "Multiply two numbers.",
        schema: z.object({
          a: z.number(),
          b: z.number(),
        }),
      }
    );

    const model = new ChatOpenAI({ model: "gpt-4o", temperature: 0 });
    const tools = [add, multiply];

    const agent = createReactAgent({
      // disable parallel tool calls
      // highlight-next-line
      llm: model.bindTools(tools, { parallel_tool_calls: false }),
      tools: tools
    });

    await agent.invoke({
      messages: [{ role: "user", content: "what's 3 + 5 and 4 * 7?" }]
    });
    ```
    :::

### 处理错误

:::python
LangGraph 通过预构建的 @[ToolNode][ToolNode] 组件提供对工具执行中的错误进行内置处理，该组件可独立使用或在预构建的 Agent 中使用。

**默认情况下**，`ToolNode` 会捕获工具执行期间引发的异常，并将其作为 `ToolMessage` 对象返回，状态指示错误。

```python
from langchain_core.messages import AIMessage
from langgraph.prebuilt import ToolNode

def multiply(a: int, b: int) -> int:
    if a == 42:
        raise ValueError("The ultimate error")
    return a * b

# 默认错误处理（默认启用）
tool_node = ToolNode([multiply])

message = AIMessage(
    content="",
    tool_calls=[{
        "name": "multiply",
        "args": {"a": 42, "b": 7},
        "id": "tool_call_id",
        "type": "tool_call"
    }]
)

result = tool_node.invoke({"messages": [message]})
```

输出：

```pycon
{'messages': [
    ToolMessage(
        content="Error: ValueError('The ultimate error')\n Please fix your mistakes.",
        name='multiply',
        tool_call_id='tool_call_id',
        status='error'
    )
]}
```

:::

:::js
LangGraph 通过预构建的 [ToolNode](https://js.langchain.com/docs/api/langgraph_prebuilt/classes/ToolNode.html) 组件提供对工具执行中的错误进行内置处理，该组件可独立使用或在预构建的 Agent 中使用。

**默认情况下**，`ToolNode` 会捕获工具执行期间引发的异常，并将其作为 `ToolMessage` 对象返回，状态指示错误。

```typescript
import { AIMessage } from "@langchain/core/messages";
import { ToolNode } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const multiply = tool(
  (input) => {
    if (input.a === 42) {
      throw new Error("The ultimate error");
    }
    return input.a * input.b;
  },
  {
    name: "multiply",
    description: "Multiply two numbers",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  }
);

// 默认错误处理（默认启用）
const toolNode = new ToolNode([multiply]);

const message = new AIMessage({
  content: "",
  tool_calls: [
    {
      name: "multiply",
      args: { a: 42, b: 7 },
      id: "tool_call_id",
      type: "tool_call",
    },
  ],
});

const result = await toolNode.invoke({ messages: [message] });
```

输出：

```
{ messages: [
  ToolMessage {
    content: "Error: The ultimate error\n Please fix your mistakes.",
    name: "multiply",
    tool_call_id: "tool_call_id",
    status: "error"
  }
]}
```

:::

#### 禁用错误处理

要直接传播异常，请禁用错误处理：

:::python

```python
tool_node = ToolNode([multiply], handle_tool_errors=False)
```

:::

:::js

```typescript
const toolNode = new ToolNode([multiply], { handleToolErrors: false });
```

:::

禁用错误处理后，工具引发的异常将向上传播，需要显式管理。

#### 自定义错误消息

通过将错误处理参数设置为字符串来提供自定义错误消息：

:::python

```python
tool_node = ToolNode(
    [multiply],
    handle_tool_errors="Can't use 42 as the first operand, please switch operands!"
)
```

示例输出：

```python
{'messages': [
    ToolMessage(
        content="Can't use 42 as the first operand, please switch operands!",
        name='multiply',
        tool_call_id='tool_call_id',
        status='error'
    )
]}
```

:::

:::js

```typescript
const toolNode = new ToolNode([multiply], {
  handleToolErrors:
    "Can't use 42 as the first operand, please switch operands!",
});
```

示例输出：

```typescript
{ messages: [
  ToolMessage {
    content: "Can't use 42 as the first operand, please switch operands!",
    name: "multiply",
    tool_call_id: "tool_call_id",
    status: "error"
  }
]}
```

:::

#### Agent 中的错误处理

:::python
预构建 Agent（`create_react_agent`）中的错误处理利用了 `ToolNode`：

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[multiply]
)

# 默认错误处理
agent.invoke({"messages": [{"role": "user", "content": "what's 42 x 7?"}]})
```

要禁用或自定义预构建 Agent 中的错误处理，请显式传递已配置的 `ToolNode`：

```python
custom_tool_node = ToolNode(
    [multiply],
    handle_tool_errors="Cannot use 42 as a first operand!"
)

agent_custom = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=custom_tool_node
)

agent_custom.invoke({"messages": [{"role": "user", "content": "what's 42 x 7?"}]})
```

:::

:::js
预构建 Agent（`createReactAgent`）中的错误处理利用了 `ToolNode`：

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { ChatAnthropic } from "@langchain/anthropic";

const agent = createReactAgent({
  llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
  tools: [multiply],
});

// 默认错误处理
await agent.invoke({
  messages: [{ role: "user", content: "what's 42 x 7?" }],
});
```

要禁用或自定义预构建 Agent 中的错误处理，请显式传递已配置的 `ToolNode`：

```typescript
const customToolNode = new ToolNode([multiply], {
  handleToolErrors: "Cannot use 42 as a first operand!",
});

const agentCustom = createReactAgent({
  llm: new ChatAnthropic({ model: "claude-3-5-sonnet-20240620" }),
  tools: customToolNode,
});

await agentCustom.invoke({
  messages: [{ role: "user", content: "what's 42 x 7?" }],
});
```

:::

### 处理大量工具

随着可用工具数量的增加，你可能希望限制 LLM 的选择范围，以减少 token 消耗并帮助管理 LLM 推理中的错误来源。

为解决此问题，你可以通过在运行时使用语义搜索检索相关工具，来动态调整模型可用的工具。

有关现成实现，请参阅 [`langgraph-bigtool`](https://github.com/langchain-ai/langgraph-bigtool) 预构建库。

## 预构建工具

### LLM 提供商工具

:::python
你可以将工具规范的字典传递给 `create_react_agent` 的 `tools` 参数，来使用来自模型提供商的预构建工具。例如，要使用 OpenAI 的 `web_search_preview` 工具：

```python
from langgraph.prebuilt import create_react_agent

agent = create_react_agent(
    model="openai:gpt-4o-mini",
    tools=[{"type": "web_search_preview"}]
)
response = agent.invoke(
    {"messages": ["What was a positive news story from today?"]}
)
```

请查阅你所使用的特定模型的文档，了解哪些工具可用及其使用方法。
:::

:::js
你可以将工具规范的字典传递给 `createReactAgent` 的 `tools` 参数，来使用来自模型提供商的预构建工具。例如，要使用 OpenAI 的 `web_search_preview` 工具：

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { ChatOpenAI } from "@langchain/openai";

const agent = createReactAgent({
  llm: new ChatOpenAI({ model: "gpt-4o-mini" }),
  tools: [{ type: "web_search_preview" }],
});

const response = await agent.invoke({
  messages: [
    { role: "user", content: "What was a positive news story from today?" },
  ],
});
```

请查阅你所使用的特定模型的文档，了解哪些工具可用及其使用方法。
:::

### LangChain 工具

此外，LangChain 还支持与 API、数据库、文件系统、Web 数据等的交互的广泛预构建工具集成。这些工具扩展了 Agent 的功能，并实现了快速开发。

:::python
你可以在 [LangChain 集成目录](https://python.langchain.com/docs/integrations/tools/) 中浏览所有可用的集成。

一些常用的工具类别包括：

-   **搜索**：Bing、SerpAPI、Tavily
-   **代码解释器**：Python REPL、Node.js REPL
-   **数据库**：SQL、MongoDB、Redis
-   **Web 数据**：网页抓取和浏览
-   **API**：OpenWeatherMap、NewsAPI 等

可以使用上面示例中显示的相同 `tools` 参数来配置和添加这些集成到 Agent 中。
:::

:::js
你可以在 [LangChain 集成目录](https://js.langchain.com/docs/integrations/tools/) 中浏览所有可用的集成。

一些常用的工具类别包括：

-   **搜索**：Tavily、SerpAPI
-   **代码解释器**：Web 浏览器、计算器
-   **数据库**：SQL、向量数据库
-   **Web 数据**：网页抓取和浏览
-   **API**：各种 API 集成

可以使用上面示例中显示的相同 `tools` 参数来配置和添加这些集成到 Agent 中。
:::