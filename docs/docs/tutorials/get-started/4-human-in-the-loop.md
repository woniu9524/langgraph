# 添加人工干预控制

Agent 可能会不可靠，并且可能需要人工输入才能成功完成任务。同样，对于某些操作，您可能希望在执行前要求人工批准，以确保一切都按预期运行。

LangGraph 的 [持久性](../../concepts/persistence.md) 层支持**人工干预**工作流，允许基于用户反馈暂停和恢复执行。此功能的主要接口是 [`interrupt`](../../how-tos/human_in_the_loop/add-human-in-the-loop.md) 函数。在节点内调用 `interrupt` 将暂停执行。可以通过传入 [Command](../../concepts/low_level.md#command) 来恢复执行，并附带来自人工的新输入。

:::python
`interrupt` 在人体工程学上类似于 Python 的内置 `input()`，[但有一些注意事项](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。
:::

:::js
`interrupt` 在人体工程学上类似于 Node.js 的内置 `readline.question()` 函数，[但有一些注意事项](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。
`interrupt` 在人体工程学上类似于 Node.js 的内置 `readline.question()` 函数，[但有一些注意事项](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。
:::

!!! note

    本教程建立在[添加内存](./3-add-memory.md)的基础上。

## 1. 添加 `human_assistance` 工具

从[为聊天机器人添加内存](./3-add-memory.md)教程的现有代码开始，将 `human_assistance` 工具添加到聊天机器人中。此工具使用 `interrupt` 来接收人工信息。

我们先选择一个聊天模型：

:::python
{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

:::

:::js

```typescript
// Add your API key here
process.env.ANTHROPIC_API_KEY = "YOUR_API_KEY";
```

:::

现在，我们可以将其与额外的工具一起集成到我们的 `StateGraph` 中：

:::python

```python hl_lines="12 19 20 21 22 23"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    # Because we will be interrupting during tool execution,
    # we disable parallel tool calling to avoid repeating any
    # tool invocations when we resume.
    assert len(message.tool_calls) <= 1
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
```

:::

:::js

```typescript hl_lines="1 7-19"
import { interrupt, MessagesZodState } from "@langchain/langgraph";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const humanAssistance = tool(
  async ({ query }) => {
    const humanResponse = interrupt({ query });
    return humanResponse.data;
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      query: z.string().describe("Human readable question for the human"),
    }),
  }
);

const searchTool = new TavilySearch({ maxResults: 2 });
const searchTool = new TavilySearch({ maxResults: 2 });
const tools = [searchTool, humanAssistance];

const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);
const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);

async function chatbot(state: z.infer<typeof MessagesZodState>) {
async function chatbot(state: z.infer<typeof MessagesZodState>) {
  const message = await llmWithTools.invoke(state.messages);


  // Because we will be interrupting during tool execution,
  // we disable parallel tool calling to avoid repeating any
  // tool invocations when we resume.
  if (message.tool_calls && message.tool_calls.length > 1) {
    throw new Error("Multiple tool calls not supported with interrupts");
  }

  return { messages: message };
}
```

:::

!!! tip

    有关人工干预工作流的更多信息和示例，请参阅[人工干预](../../concepts/human_in_the_loop.md)。

## 2. 编译图

像以前一样，我们使用检查点来编译图：

:::python

```python
memory = InMemorySaver()

graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript hl_lines="3 11"
import { StateGraph, MemorySaver, START, END } from "@langchain/langgraph";

const memory = new MemorySaver();

const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 3. 可视化图（可选）

可视化图，您将获得与以前相同的布局——只是添加了工具！

:::python

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # This requires some extra dependencies and is optional
    pass
```

:::

:::js

```typescript
import * as fs from "node:fs/promises";
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("chatbot-with-tools.png", imageBuffer);
await fs.writeFile("chatbot-with-tools.png", imageBuffer);
```

:::

![chatbot-with-tools-diagram](chatbot-with-tools.png)

## 4. 提示聊天机器人

现在，使用将调用新的 `human_assistance` 工具的问题来提示聊天机器人：

:::python

```python
user_input = "I need some expert guidance for building an AI agent. Could you request assistance for me?"
config = {"configurable": {"thread_id": "1"}}

events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

I need some expert guidance for building an AI agent. Could you request assistance for me?
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
```

:::

:::js

```typescript
import { isAIMessage } from "@langchain/core/messages";

const userInput =
  "I need some expert guidance for building an AI agent. Could you request assistance for me?";

const events = await graph.stream(
  { messages: [{ role: "user", content: userInput }] },
  { configurable: { thread_id: "1" }, streamMode: "values" }
  { configurable: { thread_id: "1" }, streamMode: "values" }
);

for await (const event of events) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
      console.log("Tool calls:", lastMessage.tool_calls);
    }
  }
}
```

```
[human]: I need some expert guidance for building an AI agent. Could you request assistance for me?
[ai]: I'll help you request human assistance for guidance on building an AI agent.
[ai]: I'll help you request human assistance for guidance on building an AI agent.
Tool calls: [
  {
    name: 'humanAssistance',
    args: {
      query: 'I would like expert guidance on building an AI agent. Could you please provide assistance with this topic?'
      query: 'I would like expert guidance on building an AI agent. Could you please provide assistance with this topic?'
    },
    id: 'toolu_01Bpxc8rFVMhSaRosS6b85Ts',
    type: 'tool_call'
    id: 'toolu_01Bpxc8rFVMhSaRosS6b85Ts',
    type: 'tool_call'
  }
]
```

:::

聊天机器人生成了一个工具调用，但然后执行被中断了。如果检查图的状态，您会发现它在工具节点处停止了：

:::python

```python
snapshot = graph.get_state(config)
snapshot.next
```

```
('tools',)
```

:::

:::js

```typescript
const snapshot = await graph.getState({ configurable: { thread_id: "1" } });
snapshot.next;
const snapshot = await graph.getState({ configurable: { thread_id: "1" } });
snapshot.next;
```

```json
["tools"]
```

:::

!!! info 附加信息

    :::python

    仔细看看 `human_assistance` 工具：

    ```python
    @tool
    def human_assistance(query: str) -> str:
        """Request assistance from a human."""
        human_response = interrupt({"query": query})
        return human_response["data"]
    ```

    与 Python 的内置 `input()` 函数类似，在工具内部调用 `interrupt` 将暂停执行。进度基于[检查点](../../concepts/persistence.md#checkpointer-libraries)进行持久化；因此，如果它使用 Postgres 进行持久化，只要数据库保持活动状态，它就可以随时恢复。在此示例中，它使用内存中的检查点进行持久化，只要 Python 内核正在运行，随时都可以恢复。
    :::

    :::js

    仔细看看 `humanAssistance` 工具：

    ```typescript hl_lines="3"
    const humanAssistance = tool(
      async ({ query }) => {
        const humanResponse = interrupt({ query });
        return humanResponse.data;
      },
      {
        name: "humanAssistance",
        description: "Request assistance from a human.",
        schema: z.object({
          query: z.string().describe("Human readable question for the human"),
        }),
      },
    );

    Take a closer look at the `humanAssistance` tool:

    ```typescript hl_lines="3"
    const humanAssistance = tool(
      async ({ query }) => {
        const humanResponse = interrupt({ query });
        return humanResponse.data;
      },
      {
        name: "humanAssistance",
        description: "Request assistance from a human.",
        schema: z.object({
          query: z.string().describe("Human readable question for the human"),
        }),
      },
    );
    ```

    在工具内部调用 `interrupt` 将暂停执行。进度基于[检查点](../../concepts/persistence.md#checkpointer-libraries)进行持久化；因此，如果它使用 Postgres 进行持久化，只要数据库保持活动状态，它就可以随时恢复。在此示例中，它使用内存中的检查点进行持久化，只要 JavaScript 运行时正在运行，随时都可以恢复。
    :::

## 5. 恢复执行

要恢复执行，请传递一个包含工具预期数据的 [`Command`](../../concepts/low_level.md#command) 对象。此数据的格式可以根据需要进行自定义。

:::python

在此示例中，使用带有 `"data"` 键的字典：

```python
human_response = (
    "We, the experts are here to help! We'd recommend you check out LangGraph to build your agent."
    " It's much more reliable and extensible than simple autonomous agents."
)

human_command = Command(resume={"data": human_response})

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

```
================================== Ai Message ==================================

[{'text': "Certainly! I'd be happy to request expert assistance for you regarding building an AI agent. To do this, I'll use the human_assistance function to relay your request. Let me do that for you now.", 'type': 'text'}, {'id': 'toolu_01ABUqneqnuHNuo1vhfDFQCW', 'input': {'query': 'A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01ABUqneqnuHNuo1vhfDFQCW)
 Call ID: toolu_01ABUqneqnuHNuo1vhfDFQCW
  Args:
    query: A user is requesting expert guidance for building an AI agent. Could you please provide some expert advice or resources on this topic?
================================= Tool Message =================================
Name: human_assistance

We, the experts are here to help! We'd recommend you check out LangGraph to build your agent. It's much more reliable and extensible than simple autonomous agents.
================================== Ai Message ==================================

Thank you for your patience. I've received some expert advice regarding your request for guidance on building an AI agent. Here's what the experts have suggested:

The experts recommend that you look into LangGraph for building your AI agent. They mention that LangGraph is a more reliable and extensible option compared to simple autonomous agents.

LangGraph is likely a framework or library designed specifically for creating AI agents with advanced capabilities. Here are a few points to consider based on this recommendation:

1. Reliability: The experts emphasize that LangGraph is more reliable than simpler autonomous agent approaches. This could mean it has better stability, error handling, or consistent performance.

2. Extensibility: LangGraph is described as more extensible, which suggests that it probably offers a flexible architecture that allows you to easily add new features or modify existing ones as your agent's requirements evolve.

3. Advanced capabilities: Given that it's recommended over "simple autonomous agents," LangGraph likely provides more sophisticated tools and techniques for building complex AI agents.
...
2. Look for tutorials or guides specifically focused on building AI agents with LangGraph.
3. Check if there are any community forums or discussion groups where you can ask questions and get support from other developers using LangGraph.

If you'd like more specific information about LangGraph or have any questions about this recommendation, please feel free to ask, and I can request further assistance from the experts.
Output is truncated. View as a scrollable element or open in a text editor. Adjust cell output settings...
```

:::

:::js
在此示例中，使用带有 `"data"` 键的对象：

```typescript
import { Command } from "@langchain/langgraph";

const humanResponse =
  "We, the experts are here to help! We'd recommend you check out LangGraph to build your agent." +
  " It's much more reliable and extensible than simple autonomous agents.";
(" It's much more reliable and extensible than simple autonomous agents.");

const humanCommand = new Command({ resume: { data: humanResponse } });

const resumeEvents = await graph.stream(humanCommand, {
  configurable: { thread_id: "1" },
  streamMode: "values",
});
const resumeEvents = await graph.stream(humanCommand, {
  configurable: { thread_id: "1" },
  streamMode: "values",
});

for await (const event of resumeEvents) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);
    const lastMessage = event.messages.at(-1);
    console.log(`[${lastMessage?.getType()}]: ${lastMessage?.text}`);
  }
}
```

```
[tool]: We, the experts are here to help! We'd recommend you check out LangGraph to build your agent. It's much more reliable and extensible than simple autonomous agents.
[ai]: Thank you for your patience. I've received some expert advice regarding your request for guidance on building an AI agent. Here's what the experts have suggested:

The experts recommend that you look into LangGraph for building your AI agent. They mention that LangGraph is a more reliable and extensible option compared to simple autonomous agents.

LangGraph is likely a framework or library designed specifically for creating AI agents with advanced capabilities. Here are a few points to consider based on this recommendation:

1. Reliability: The experts emphasize that LangGraph is more reliable than simpler autonomous agent approaches. This could mean it has better stability, error handling, or consistent performance.

2. Extensibility: LangGraph is described as more extensible, which suggests that it probably offers a flexible architecture that allows you to easily add new features or modify existing ones as your agent's requirements evolve.

3. Advanced capabilities: Given that it's recommended over "simple autonomous agents," LangGraph likely provides more sophisticated tools and techniques for building complex AI agents.

...
```

:::

输入已收到并作为工具消息处理。查看此调用的 [LangSmith 跟踪](https://smith.langchain.com/public/9f0f87e3-56a7-4dde-9c76-b71675624e91/r)以了解上述调用中执行的确切工作。请注意，状态在第一个步骤中加载，以便我们的聊天机器人可以从上次中断的地方继续。

**恭喜！** 您已使用 `interrupt` 为聊天机器人添加了人工干预执行功能，从而在需要时可以进行人工监督和干预。这为构建 AI 系统开辟了新的用户界面潜力。由于您已经添加了**检查点**，只要底层持久化层正在运行，图就可以**无限期**地暂停，并像什么都没发生一样随时恢复。

查看本教程中的代码片段：

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.tools import tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

@tool
def human_assistance(query: str) -> str:
    """Request assistance from a human."""
    human_response = interrupt({"query": query})
    return human_response["data"]

tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1)
    return {"messages": [message]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=tools)
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")

memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript
import {
  interrupt,
  MessagesZodState,
  StateGraph,
  MemorySaver,
  START,
  END,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { isAIMessage } from "@langchain/core/messages";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import {
  interrupt,
  MessagesZodState,
  StateGraph,
  MemorySaver,
  START,
  END,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { isAIMessage } from "@langchain/core/messages";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const humanAssistance = tool(
  async ({ query }) => {
    const humanResponse = interrupt({ query });
    return humanResponse.data;
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      query: z.string().describe("Human readable question for the human"),
    }),
  }
);
const humanAssistance = tool(
  async ({ query }) => {
    const humanResponse = interrupt({ query });
    return humanResponse.data;
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      query: z.string().describe("Human readable question for the human"),
    }),
  }
);

const searchTool = new TavilySearch({ maxResults: 2 });
const searchTool = new TavilySearch({ maxResults: 2 });
const tools = [searchTool, humanAssistance];

const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);
const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);

const chatbot = async (state: z.infer<typeof MessagesZodState>) => {
const chatbot = async (state: z.infer<typeof MessagesZodState>) => {
  const message = await llmWithTools.invoke(state.messages);

  // Because we will be interrupting during tool execution,
  // we disable parallel tool calling to avoid repeating any
  // tool invocations when we resume.

  // Because we will be interrupting during tool execution,
  // we disable parallel tool calling to avoid repeating any
  // tool invocations when we resume.
  if (message.tool_calls && message.tool_calls.length > 1) {
    throw new Error("Multiple tool calls not supported with interrupts");
  }

  return { messages: message };

  return { messages: message };
};

const memory = new MemorySaver();

const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });

const graph = new StateGraph(MessagesZodState)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 下一步

到目前为止，教程示例一直依赖于一个简单的状态，其中包含一个条目：消息列表。您可以使用这个简单的状态做很多事情，但如果您想在不依赖消息列表的情况下定义复杂行为，您可以[向状态添加额外字段](./5-customize-state.md)。