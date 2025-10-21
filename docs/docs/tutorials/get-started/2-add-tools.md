# 集成工具

为了处理您的聊天机器人无法“凭空”回答的查询，请集成网络搜索工具。聊天机器人可以使用此工具查找相关信息并提供更好的响应。

!!! note

    本教程建立在 [构建一个基础聊天机器人](./1-build-basic-chatbot.md) 的基础上。

## 先决条件

在开始本教程之前，请确保您拥有以下内容：

:::python

- [Tavily Search Engine](https://python.langchain.com/docs/integrations/tools/tavily_search/) 的 API 密钥。

:::

:::js

- [Tavily Search Engine](https://js.langchain.com/docs/integrations/tools/tavily_search/) 的 API 密钥。

:::

## 1. 安装搜索引擎

:::python
安装使用 [Tavily Search Engine](https://python.langchain.com/docs/integrations/tools/tavily_search/) 的所需依赖：

```bash
pip install -U langchain-tavily
```

:::

:::js
安装使用 [Tavily Search Engine](https://docs.tavily.com/) 的所需依赖：

=== "npm"

    ```bash
    npm install @langchain/tavily
    ```

=== "yarn"

    ```bash
    yarn add @langchain/tavily
    ```

=== "pnpm"

    ```bash
    pnpm add @langchain/tavily
    ```

=== "bun"

    ```bash
    bun add @langchain/tavily
    ```

:::

## 2. 配置您的环境

使用您的搜索引擎 API 密钥配置您的环境：

:::python
```python
import os

os.environ["TAVILY_API_KEY"] = "tvly-..."
```
:::

:::js

```typescript
process.env.TAVILY_API_KEY = "tvly-...";
```

:::

## 3. 定义工具

定义网络搜索工具：

:::python

```python
from langchain_tavily import TavilySearch

tool = TavilySearch(max_results=2)
tools = [tool]
tool.invoke("What's a 'node' in LangGraph?")
```

:::

:::js

```typescript
import { TavilySearch } from "@langchain/tavily";

const tool = new TavilySearch({ maxResults: 2 });
const tools = [tool];

await tool.invoke({ query: "What's a 'node' in LangGraph?" });
```

:::

结果是我们的聊天机器人可以用来回答问题的页面摘要：

:::python

```
{'query': "What's a 'node' in LangGraph?",
'follow_up_questions': None,
'answer': None,
'images': [],
'results': [{'title': "Introduction to LangGraph: A Beginner's Guide - Medium",
'url': 'https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141',
'content': 'Stateful Graph: LangGraph revolves around the concept of a stateful graph, where each node in the graph represents a step in your computation, and the graph maintains a state that is passed around and updated as the computation progresses. LangGraph supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph. We define nodes for classifying the input, handling greetings, and handling search queries. def classify_input_node(state): LangGraph is a versatile tool for building complex, stateful applications with LLMs. By understanding its core concepts and working through simple examples, beginners can start to leverage its power for their projects. Remember to pay attention to state management, conditional edges, and ensuring there are no dead-end nodes in your graph.',
'score': 0.7065353,
'raw_content': None},
{'title': 'LangGraph Tutorial: What Is LangGraph and How to Use It?',
'url': 'https://www.datacamp.com/tutorial/langgraph-tutorial',
'content': 'LangGraph is a library within the LangChain ecosystem that provides a framework for defining, coordinating, and executing multiple LLM agents (or chains) in a structured and efficient manner. By managing the flow of data and the sequence of operations, LangGraph allows developers to focus on the high-level logic of their applications rather than the intricacies of agent coordination. Whether you need a chatbot that can handle various types of user requests or a multi-agent system that performs complex tasks, LangGraph provides the tools to build exactly what you need. LangGraph significantly simplifies the development of complex LLM applications by providing a structured framework for managing state and coordinating agent interactions.',
'score': 0.5008063,
'raw_content': None}],
'response_time': 1.38}
```

:::

:::js

```json
{
  "query": "What's a 'node' in LangGraph?",
  "follow_up_questions": null,
  "answer": null,
  "images": [],
  "results": [
    {
      "url": "https://blog.langchain.dev/langgraph/",
      "title": "LangGraph - LangChain Blog",
      "content": "TL;DR: LangGraph is module built on top of LangChain to better enable creation of cyclical graphs, often needed for agent runtimes. This state is updated by nodes in the graph, which return operations to attributes of this state (in the form of a key-value store). After adding nodes, you can then add edges to create the graph. An example of this may be in the basic agent runtime, where we always want the model to be called after we call a tool. The state of this graph by default contains concepts that should be familiar to you if you've used LangChain agents: `input`, `chat_history`, `intermediate_steps` (and `agent_outcome` to represent the most recent agent outcome)",
      "score": 0.7407191,
      "raw_content": null
    },
    {
      "url": "https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141",
      "title": "Introduction to LangGraph: A Beginner's Guide - Medium",
      "content": "*   **Stateful Graph:** LangGraph revolves around the concept of a stateful graph, where each node in the graph represents a step in your computation, and the graph maintains a state that is passed around and updated as the computation progresses. LangGraph supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph. Image 10: Introduction to AI Agent with LangChain and LangGraph: A Beginner’s Guide Image 18: How to build LLM Agent with LangGraph — StateGraph and Reducer Image 20: Simplest Graphs using LangGraph Framework Image 24: Building a ReAct Agent with Langgraph: A Step-by-Step Guide Image 28: Building an Agentic RAG with LangGraph: A Step-by-Step Guide",
      "score": 0.65279555,
      "raw_content": null
    }
  ],
  "response_time": 1.34
}
```

:::

## 4. 定义图

:::python
对于您在 [第一个教程](./1-build-basic-chatbot.md) 中创建的 `StateGraph`，在 LLM 上调用 `bind_tools`。这能让 LLM 知道如果它想要使用搜索引擎，需要使用正确的 JSON 格式。
:::

:::js
对于您在 [第一个教程](./1-build-basic-chatbot.md) 中创建的 `StateGraph`，在 LLM 上调用 `bindTools`。这能让 LLM 知道如果它想要使用搜索引擎，需要使用正确的 JSON 格式。
:::

让我们先选择我们的 LLM：

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
import { ChatAnthropic } from "@langchain/anthropic";

const llm = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });
```

:::

现在我们可以将其集成到 `StateGraph` 中：

:::python

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

# 修改：告知 LLM 可以调用的工具
# highlight-next-line
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)
```

:::

:::js

```typescript hl_lines="7-8"
import { StateGraph, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const chatbot = async (state: z.infer<typeof State>) => {
  // 修改：告知 LLM 可以调用的工具
  const llmWithTools = llm.bindTools(tools);

  return { messages: [await llmWithTools.invoke(state.messages)] };
};
```

:::

## 5. 创建一个运行工具的函数

:::python

现在，创建一个函数来运行工具（如果它们被调用）。通过将工具添加到名为 `BasicToolNode` 的新节点来完成此操作，该节点会检查状态中的最新消息，并在消息包含 `tool_calls` 时调用工具。它依赖于 LLM 的 `tool_calling` 支持，这在 Anthropic、OpenAI、Google Gemini 以及其他许多 LLM 提供商中都可用。

```python
import json

from langchain_core.messages import ToolMessage


class BasicToolNode:
    """一个运行 AIMessage 中请求的工具的节点。"""

    def __init__(self, tools: list) -> None:
        self.tools_by_name = {tool.name: tool for tool in tools}

    def __call__(self, inputs: dict):
        if messages := inputs.get("messages", []):
            message = messages[-1]
        else:
            raise ValueError("No message found in input")
        outputs = []
        for tool_call in message.tool_calls:
            tool_result = self.tools_by_name[tool_call["name"]].invoke(
                tool_call["args"]
            )
            outputs.append(
                ToolMessage(
                    content=json.dumps(tool_result),
                    name=tool_call["name"],
                    tool_call_id=tool_call["id"],
                )
            )
        return {"messages": outputs}


tool_node = BasicToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)
```

!!! note

    如果您将来不想自己构建此功能，可以使用 LangGraph 提供的预构建 [ToolNode](https://langchain-ai.github.io/langgraph/reference/agents/#langgraph.prebuilt.tool_node.ToolNode)。

:::

:::js

现在，创建一个函数来运行工具（如果它们被调用）。通过将工具添加到名为 `"tools"` 的新节点来完成此操作，该节点会检查状态中的最新消息，并在消息包含 `tool_calls` 时调用工具。它依赖于 LLM 的工具调用支持，这在 Anthropic、OpenAI、Google Gemini 以及其他许多 LLM 提供商中都可用。

```typescript
import type { StructuredToolInterface } from "@langchain/core/tools";
import { isAIMessage, ToolMessage } from "@langchain/core/messages";

function createToolNode(tools: StructuredToolInterface[]) {
  const toolByName: Record<string, StructuredToolInterface> = {};
  for (const tool of tools) {
    toolByName[tool.name] = tool;
  }

  return async (inputs: z.infer<typeof State>) => {
    const { messages } = inputs;
    if (!messages || messages.length === 0) {
      throw new Error("No message found in input");
    }

    const message = messages.at(-1);
    if (!message || !isAIMessage(message) || !message.tool_calls) {
      throw new Error("Last message is not an AI message with tool calls");
    }

    const outputs: ToolMessage[] = [];
    for (const toolCall of message.tool_calls) {
      if (!toolCall.id) throw new Error("Tool call ID is required");

      const tool = toolByName[toolCall.name];
      if (!tool) throw new Error(`Tool ${toolCall.name} not found`);

      const result = await tool.invoke(toolCall.args);

      outputs.push(
        new ToolMessage({
          content: JSON.stringify(result),
          name: toolCall.name,
          tool_call_id: toolCall.id,
        })
      );
    }

    return { messages: outputs };
  };
}
```

!!! note

    如果您将来不想自己构建此功能，可以使用 LangGraph 提供的预构建 [ToolNode](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph_prebuilt.ToolNode.html)。

:::

## 6. 定义 `conditional_edges`

添加了工具节点后，现在可以定义 `conditional_edges`。

**边（Edges）** 将控制流从一个节点路由到下一个节点。**条件边（Conditional Edges）** 从单个节点开始，通常包含“if”语句，用于根据当前图状态路由到不同的节点。这些函数接收当前的图 `state` 并返回一个字符串或字符串列表，指示下一个要调用的节点（或节点）。

:::python
接下来，定义一个名为 `route_tools` 的路由函数，该函数会检查聊天机器人输出中的 `tool_calls`。通过调用 `add_conditional_edges` 将此函数提供给图，这会告诉图，每当 `chatbot` 节点完成时，就检查此函数以确定下一个去向。
:::

:::js
接下来，定义一个名为 `routeTools` 的路由函数，该函数会检查聊天机器人输出中的 `tool_calls`。通过调用 `addConditionalEdges` 将此函数提供给图，这会告诉图，每当 `chatbot` 节点完成时，就检查此函数以确定下一个去向。
:::

该条件将路由到 `tools`（如果存在工具调用）或 `END`（如果不存在）。因为条件可以返回 `END`，所以这次您无需显式设置 `finish_point`。

:::python

```python
def route_tools(
    state: State,
):
    """
    在条件边中使用，如果最后一条消息包含工具调用，则路由到 ToolNode。
    否则，路由到 END。
    """
    if isinstance(state, list):
        ai_message = state[-1]
    elif messages := state.get("messages", []):
        ai_message = messages[-1]
    else:
        raise ValueError(f"No messages found in input state to tool_edge: {state}")
    if hasattr(ai_message, "tool_calls") and len(ai_message.tool_calls) > 0:
        return "tools"
    return END


# `tools_condition` 函数在聊天机器人请求使用工具时返回“tools”，在可以直接响应时返回“END”。
# 这个条件路由定义了主要的代理循环。
graph_builder.add_conditional_edges(
    "chatbot",
    route_tools,
    # 下面的字典允许您告诉图将条件输出解释为特定节点
    # 它默认为恒等函数，但如果您
    # 想使用一个名称不同于“tools”的节点，
    # 您可以将字典的值更新为其他内容
    # 例如：“tools”: “my_tools”
    {"tools": "tools", END: END},
)
# 任何时候调用工具，我们都会返回到聊天机器人以决定下一步
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
graph = graph_builder.compile()
```

!!! note

    您可以使用预构建的 [tools_condition](https://langchain-ai.github.io/langgraph/reference/prebuilt/#tools_condition) 来代替，这样更简洁。

:::

:::js

```typescript
import { END, START } from "@langchain/langgraph";

const routeTools = (state: z.infer<typeof State>) => {
  /**
   * 用作条件边，如果最后一条消息包含工具调用，则路由到 ToolNode。
   */
  const lastMessage = state.messages.at(-1);
  if (
    lastMessage &&
    isAIMessage(lastMessage) &&
    lastMessage.tool_calls?.length
  ) {
    return "tools";
  }

  /** 否则，路由到 END。 */
  return END;
};

const graph = new StateGraph(State)
  .addNode("chatbot", chatbot)

  // `routeTools` 函数在聊天机器人请求使用工具时返回“tools”，在可以直接响应时返回“END”。
  // 这个条件路由定义了主要的代理循环。
  .addNode("tools", createToolNode(tools))

  // 启动图，从聊天机器人开始
  .addEdge(START, "chatbot")

  // `routeTools` 函数在聊天机器人请求使用工具时返回“tools”，在可以直接响应时返回“END”。
  .addConditionalEdges("chatbot", routeTools, ["tools", END])

  // 任何时候调用工具，我们都需要返回到聊天机器人
  .addEdge("tools", "chatbot")
  .compile();
```

!!! note

    您可以使用预构建的 [toolsCondition](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph_prebuilt.toolsCondition.html) 来代替，这样更简洁。

:::

## 7. 可视化图（可选）

:::python
您可以使用 `get_graph` 方法和其中一个“draw”方法（如 `draw_ascii` 或 `draw_png`）来可视化图。`draw` 方法每个都需要额外的依赖项。

```python
from IPython.display import Image, display

try:
    display(Image(graph.get_graph().draw_mermaid_png()))
except Exception:
    # 这需要一些额外的依赖项，并且是可选的
    pass
```

:::

:::js
您可以使用 `getGraph` 方法并使用 `drawMermaidPng` 方法渲染图来可视化图。

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("chatbot-with-tools.png", imageBuffer);
```

:::

![chatbot-with-tools-diagram](chatbot-with-tools.png)

## 8. 询问机器人问题

现在您可以问聊天机器人它训练数据之外的问题了：

:::python

```python
def stream_graph_updates(user_input: str):
    for event in graph.stream({"messages": [{"role": "user", "content": user_input}]}):
        for value in event.values():
            print("Assistant:", value["messages"][-1].content)

while True:
    try:
        user_input = input("User: ")
        if user_input.lower() in ["quit", "exit", "q"]:
            print("Goodbye!")
            break

        stream_graph_updates(user_input)
    except:
        # fallback if input() is not available
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```

```
Assistant: [{'text': "To provide you with accurate and up-to-date information about LangGraph, I'll need to search for the latest details. Let me do that for you.", 'type': 'text'}, {'id': 'toolu_01Q588CszHaSvvP2MxRq9zRD', 'input': {'query': 'LangGraph AI tool information'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Assistant: [{"url": "https://www.langchain.com/langgraph", "content": "LangGraph sets the foundation for how we can build and scale AI workloads \u2014 from conversational agents, complex task automation, to custom LLM-backed experiences that 'just work'. The next chapter in building complex production-ready features with LLMs is agentic, and with LangGraph and LangSmith, LangChain delivers an out-of-the-box solution ..."}, {"url": "https://github.com/langchain-ai/langgraph", "content": "Overview. LangGraph is a library for building stateful, multi-actor applications with LLMs, used to create agent and multi-agent workflows. Compared to other LLM frameworks, it offers these core benefits: cycles, controllability, and persistence. LangGraph allows you to define flows that involve cycles, essential for most agentic architectures ..."}]
Assistant: Based on the search results, I can provide you with information about LangGraph:

1. Purpose:
   LangGraph is a library designed for building stateful, multi-actor applications with Large Language Models (LLMs). It's particularly useful for creating agent and multi-agent workflows.

2. Developer:
   LangGraph is developed by LangChain, a company known for its tools and frameworks in the AI and LLM space.

3. Key Features:
   - Cycles: LangGraph allows the definition of flows that involve cycles, which is essential for most agentic architectures.
   - Controllability: It offers enhanced control over the application flow.
   - Persistence: The library provides ways to maintain state and persistence in LLM-based applications.

4. Use Cases:
   LangGraph can be used for various applications, including:
   - Conversational agents
   - Complex task automation
   - Custom LLM-backed experiences

5. Integration:
   LangGraph works in conjunction with LangSmith, another tool by LangChain, to provide an out-of-the-box solution for building complex, production-ready features with LLMs.

6. Significance:
...
   LangGraph is noted to offer unique benefits compared to other LLM frameworks, particularly in its ability to handle cycles, provide controllability, and maintain persistence.

LangGraph appears to be a significant tool in the evolving landscape of LLM-based application development, offering developers new ways to create more complex, stateful, and interactive AI systems.
Goodbye!
```

:::

:::js

```typescript
import readline from "node:readline/promises";

const prompt = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

async function generateText(content: string) {
  const stream = await graph.stream(
    { messages: [{ type: "human", content }] },
    { streamMode: "values" }
  );

  for await (const event of stream) {
    const lastMessage = event.messages.at(-1);

    if (lastMessage?.getType() === "ai" || lastMessage?.getType() === "tool") {
      console.log(`Assistant: ${lastMessage?.text}`);
    }
  }
}

while (true) {
  const human = await prompt.question("User: ");
  if (["quit", "exit", "q"].includes(human.trim())) break;
  await generateText(human || "What do you know about LangGraph?");
}

prompt.close();
```

```
User: What do you know about LangGraph?
Assistant: I'll search for the latest information about LangGraph for you.
Assistant: [{"title":"Introduction to LangGraph: A Beginner's Guide - Medium","url":"https://medium.com/@cplog/introduction-to-langgraph-a-beginners-guide-14f9be027141","content":"..."}]
Assistant: Based on the search results, I can provide you with information about LangGraph:

LangGraph is a library within the LangChain ecosystem designed for building stateful, multi-actor applications with Large Language Models (LLMs). Here are the key aspects:

**Core Purpose:**
- LangGraph is specifically designed for creating agent and multi-agent workflows
- It provides a framework for defining, coordinating, and executing multiple LLM agents in a structured manner

**Key Features:**
1. **Stateful Graph Architecture**: LangGraph revolves around a stateful graph where each node represents a step in computation, and the graph maintains state that is passed around and updated as the computation progresses

2. **Conditional Edges**: It supports conditional edges, allowing you to dynamically determine the next node to execute based on the current state of the graph

3. **Cycles**: Unlike other LLM frameworks, LangGraph allows you to define flows that involve cycles, which is essential for most agentic architectures

4. **Controllability**: It offers enhanced control over the application flow

5. **Persistence**: The library provides ways to maintain state and persistence in LLM-based applications

**Use Cases:**
- Conversational agents
- Complex task automation
- Custom LLM-backed experiences
- Multi-agent systems that perform complex tasks

**Benefits:**
LangGraph allows developers to focus on the high-level logic of their applications rather than the intricacies of agent coordination, making it easier to build complex, production-ready features with LLMs.

This makes LangGraph a significant tool in the evolving landscape of LLM-based application development.
```

:::

## 9. 使用预构建组件

为了方便使用，请调整您的代码，将以下内容替换为 LangGraph 的预构建组件。这些组件内置了并行 API 执行等功能。

:::python

- `BasicToolNode` 被预构建的 [ToolNode](https://langchain-ai.github.io/langgraph/reference/prebuilt/#toolnode) 替换。
- `route_tools` 被预构建的 [tools_condition](https://langchain-ai.github.io/langgraph/reference/prebuilt/#tools_condition) 替换。

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python hl_lines="25 30"
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition

class State(TypedDict):
    messages: Annotated[list, add_messages]

graph_builder = StateGraph(State)

tool = TavilySearch(max_results=2)
tools = [tool]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    return {"messages": [llm_with_tools.invoke(state["messages"])]}

graph_builder.add_node("chatbot", chatbot)

tool_node = ToolNode(tools=[tool])
graph_builder.add_node("tools", tool_node)

graph_builder.add_conditional_edges(
    "chatbot",
    tools_condition,
)
# 任何时候调用工具，我们都会返回到聊天机器人以决定下一步
graph_builder.add_edge("tools", "chatbot")
graph_builder.add_edge(START, "chatbot")
graph = graph_builder.compile()
```

:::

:::js

- `createToolNode` 被预构建的 [ToolNode](https://langchain-ai.github.io/langgraphjs/reference/classes/langgraph_prebuilt.ToolNode.html) 替换。
- `routeTools` 被预构建的 [toolsCondition](https://langchain-ai.github.io/langgraphjs/reference/functions/langgraph_prebuilt.toolsCondition.html) 替换。

```typescript
import { TavilySearch } from "@langchain/tavily";
import { ChatOpenAI } from "@langchain/openai";
import { StateGraph, START, MessagesZodState, END } from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const tools = [new TavilySearch({ maxResults: 2 })];

const llm = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools(tools);

const graph = new StateGraph(State)
  .addNode("chatbot", async (state) => ({
    messages: [await llm.invoke(state.messages)],
  }))
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile();
```

:::

**恭喜！** 您已经在 LangGraph 中创建了一个能够使用搜索引擎检索最新信息的会话代理。现在它可以处理更广泛的用户查询了。

:::python

要检查代理刚刚采取的所有步骤，请查看此 [LangSmith 跟踪](https://smith.langchain.com/public/4fbd7636-25af-4638-9587-5a02fdbb0172/r)。

:::

## 下一步

聊天机器人无法自主记住过去的交互，这限制了它进行连贯、多轮对话的能力。在下一部分中，您将 [添加**记忆**](./3-add-memory.md) 来解决这个问题。