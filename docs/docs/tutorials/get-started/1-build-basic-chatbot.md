# 构建一个基础聊天机器人

在本教程中，你将构建一个基础聊天机器人。该聊天机器人将作为后续教程系列的基础，你将在其中逐步添加更复杂的功能，并在此过程中了解 LangGraph 的核心概念。让我们开始吧！🌟

## 前提条件

在开始本教程之前，请确保你能够访问支持
工具调用功能的 LLM（大语言模型），例如 [OpenAI](https://platform.openai.com/api-keys)、
[Anthropic](https://console.anthropic.com/settings/keys) 或
[Google Gemini](https://ai.google.dev/gemini-api/docs/api-key)。

## 1. 安装包

安装所需的包：

:::python

```bash
pip install -U langraph langsmith
```

:::

:::js
=== "npm"

    ```bash
    npm install @langchain/langgraph @langchain/core zod
    ```

=== "yarn"

    ```bash
    yarn add @langchain/langgraph @langchain/core zod
    ```

=== "pnpm"

    ```bash
    pnpm add @langchain/langgraph @langchain/core zod
    ```

=== "bun"

    ```bash
    bun add @langchain/langgraph @langchain/core zod
    ```

:::

!!! tip

    注册 LangSmith，以便快速发现问题并改善 LangGraph 项目的性能。LangSmith 允许你使用跟踪数据来调试、测试和监控使用 LangGraph 构建的 LLM 应用。有关如何开始的更多信息，请参阅 [LangSmith 文档](https://docs.smith.langchain.com)。

## 2. 创建 `StateGraph`

现在，你可以使用 LangGraph 创建一个基础聊天机器人。这个聊天机器人将直接响应用户的消息。

首先创建一个 `StateGraph`。`StateGraph` 对象将我们的聊天机器人结构定义为“状态机”。我们将添加 `nodes` 来表示聊天机器人可以调用的 LLM 和函数，以及 `edges` 来指定机器人如何在这些函数之间进行转换。

:::python

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    # Messages 的类型是 "list"。
    # 类型注解中的 `add_messages` 函数定义了如何更新此状态键
    # （在本例中，它是将消息添加到列表中，而不是覆盖它们）
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)
```

:::

:::js

```typescript
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State).compile();
```

:::

我们的图现在可以处理两个关键任务：

1. 每个 `node` 都可以接收当前 `State` 作为输入，并输出对状态的更新。
2. 由于使用了预构建的 reducer 函数，`messages` 的更新将被追加到现有列表中，而不是覆盖它们。

!!! tip "概念"

    在定义图时，第一步是定义其 `State`。`State` 包含图的模式和处理状态更新的[reducer 函数](https://langchain-ai.github.io/langgraph/concepts/low_level/#reducers)。在我们的示例中，`State` 是一个只包含一个键的模式：`messages`。reducer 函数用于将新消息追加到列表中，而不是覆盖它。没有 reducer 注解的键将覆盖先前的值。

    要了解更多关于状态、reducer 和相关概念的信息，请参阅 [LangGraph 参考文档](https://langchain-ai.github.io/langgraph/reference/graphs/#langgraph.graph.message.add_messages)。

## 3. 添加一个节点

接下来，添加一个“`chatbot`”节点。**节点**代表工作单元，通常是常规函数。

让我们先选择一个聊天模型：

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
import { ChatOpenAI } from "@langchain/openai";
// 或者 import { ChatAnthropic } from "@langchain/anthropic";

const llm = new ChatOpenAI({
  model: "gpt-4o",
  temperature: 0,
});
```

:::

我们现在可以将聊天模型纳入一个简单的节点：

:::python

```python

def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一个参数是唯一的节点名称
# 第二个参数是在节点被调用时将执行的函数或对象。
graph_builder.add_node("chatbot", chatbot)
```

:::

:::js

```typescript hl_lines="7-9"
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .compile();
```

:::

**请注意** `chatbot` 节点函数如何接收当前 `State` 作为输入，并返回一个包含更新后的 `messages` 列表的字典，该列表位于“messages”键下。这是所有 LangGraph 节点函数的基本模式。

:::python
我们 `State` 中的 `add_messages` 函数会将 LLM 的响应消息追加到状态中已有的任何消息后面。
:::

:::js
`MessagesZodState` 中使用的 `addMessages` 函数会将 LLM 的响应消息追加到状态中已有的任何消息后面。
:::

## 4. 添加一个 `entry` 点

添加一个 `entry` 点来告诉图每次运行时**从哪里开始工作**：

:::python

```python
graph_builder.add_edge(START, "chatbot")
```

:::

:::js

```typescript hl_lines="10"
import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .compile();
```

:::

## 5. 添加一个 `exit` 点

添加一个 `exit` 点来指示**图应该在哪里结束执行**。这对于更复杂的流程很有用，但即使在这个简单的图中，添加一个结束节点也能提高清晰度。

:::python

```python
graph_builder.add_edge("chatbot", END)
```

:::

:::js

```typescript hl_lines="11"
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .addEdge("chatbot", END)
  .compile();
```

:::

这会告诉图在运行完 `chatbot` 节点后终止。

## 6. 编译图

在运行图之前，我们需要编译它。我们可以通过调用图构建器上的 `compile()` 来实现。这将创建一个 `CompiledGraph`，我们可以在其状态上调用它。

:::python

```python
graph = graph_builder.compile()
```

:::

:::js

```typescript hl_lines="12"
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .addEdge("chatbot", END)
  .compile();
```

:::

## 7. 可视化图（可选）

:::python
你可以使用 `get_graph` 方法和一个“draw”方法（如 `draw_ascii` 或 `draw_png`）来可视化图。`draw` 方法需要额外的依赖项。

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
你可以使用 `getGraph` 方法可视化图，并使用 `drawMermaidPng` 方法渲染图。

```typescript
import * as fs from "node:fs/promises";

const drawableGraph = await graph.getGraphAsync();
const image = await drawableGraph.drawMermaidPng();
const imageBuffer = new Uint8Array(await image.arrayBuffer());

await fs.writeFile("basic-chatbot.png", imageBuffer);
```

:::

![basic chatbot diagram](basic-chatbot.png)

## 8. 运行聊天机器人

现在运行聊天机器人！

!!! tip

    你随时可以通过输入 `quit`、`exit` 或 `q` 来退出聊天循环。

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
        # 如果 input() 不可用，则回退
        user_input = "What do you know about LangGraph?"
        print("User: " + user_input)
        stream_graph_updates(user_input)
        break
```

:::

:::js

```typescript
import { HumanMessage } from "@langchain/core/messages";

async function streamGraphUpdates(userInput: string) {
  const stream = await graph.stream({
    messages: [new HumanMessage(userInput)],
  });

import * as readline from "node:readline/promises";
import { StateGraph, MessagesZodState, START, END } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { z } from "zod";

const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State)
  .addNode("chatbot", async (state: z.infer<typeof State>) => {
    return { messages: [await llm.invoke(state.messages)] };
  })
  .addEdge(START, "chatbot")
  .addEdge("chatbot", END)
  .compile();

async function generateText(content: string) {
  const stream = await graph.stream(
    { messages: [{ type: "human", content }] },
    { streamMode: "values" }
  );

  for await (const event of stream) {
    for (const value of Object.values(event)) {
      console.log(
        "Assistant:",
        value.messages[value.messages.length - 1].content
      );
    const lastMessage = event.messages.at(-1);
    if (lastMessage?.getType() === "ai") {
      console.log(`Assistant: ${lastMessage.text}`);
    }
  }
}

const prompt = readline.createInterface({
  input: process.stdin,
  output: process.stdout,
});

while (true) {
  const human = await prompt.question("User: ");
  if (["quit", "exit", "q"].includes(human.trim())) break;
  await generateText(human || "What do you know about LangGraph?");
}

prompt.close();
```

:::

```
Assistant: LangGraph 是一个库，旨在帮助使用语言模型构建有状态的多代理应用程序。它提供了用于创建工作流和状态机的工具，以协调多个 AI 代理或语言模型交互。LangGraph 构建在 LangChain 之上，利用其组件的同时增加了基于图的协调功能。它特别适用于开发比简单查询-响应交互更复杂、更有状态的 AI 应用程序。
```

:::python

```
Goodbye!
```

:::

**恭喜！**你已经使用 LangGraph 构建了你的第一个聊天机器人。这个机器人可以通过用户的输入进行基本的对话，并使用 LLM 生成响应。你可以查看上面的调用的 [LangSmith Trace](https://smith.langchain.com/public/7527e308-9502-4894-b347-f34385740d5a/r)。

:::python

本教程的完整代码如下：

```python
from typing import Annotated

from langchain.chat_models import init_chat_model
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]


graph_builder = StateGraph(State)


llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")


def chatbot(state: State):
    return {"messages": [llm.invoke(state["messages"])]}


# 第一个参数是唯一的节点名称
# 第二个参数是在节点被调用时将执行的函数或对象。
graph_builder.add_node("chatbot", chatbot)
graph_builder.add_edge(START, "chatbot")
graph_builder.add_edge("chatbot", END)
graph = graph_builder.compile()
```

:::

:::js

```typescript
import { StateGraph, START, END, MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";
import { ChatOpenAI } from "@langchain/openai";

const llm = new ChatOpenAI({
  model: "gpt-4o",
  temperature: 0,
});

const State = z.object({ messages: MessagesZodState.shape.messages });

const graph = new StateGraph(State);
  // 第一个参数是唯一的节点名称
  // 第二个参数是在节点被调用时将执行的函数或对象。
  .addNode("chatbot", async (state) => {
    return { messages: [await llm.invoke(state.messages)] };
  });
  .addEdge(START, "chatbot");
  .addEdge("chatbot", END)
  .compile();
```

:::

## 后续步骤

你可能已经注意到，该机器人的知识仅限于其训练数据。在下一部分中，我们将[添加一个网络搜索工具](./2-add-tools.md)来扩展机器人的知识并使其更强大。