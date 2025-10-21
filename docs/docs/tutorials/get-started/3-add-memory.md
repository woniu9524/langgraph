# 添加记忆

聊天机器人现在可以使用 [工具](./2-add-tools.md) 来回答用户的问题，但它不记得之前交互的上下文。这限制了它进行连贯的多轮对话的能力。

LangGraph 通过**持久化检查点（persistent checkpointing）**解决了这个问题。如果您在编译图表时提供了一个 `checkpointer`，并在调用图表时提供了一个 `thread_id`，LangGraph 会在每一步之后自动保存状态。当您使用相同的 `thread_id` 再次调用图表时，图表会加载其保存的状态，使聊天机器人能够接续之前的进度。

我们稍后会看到，**检查点** 比简单的聊天记忆要强大得多——它可以让您随时保存和恢复复杂的状态，用于错误恢复、人工介入工作流、时间旅行交互等等。但首先，让我们添加检查点以启用多轮对话。

!!! note

    本教程建立在 [添加工具](./2-add-tools.md) 的基础上。

## 1. 创建一个 `MemorySaver` 检查点

创建一个 `MemorySaver` 检查点：

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver

memory = InMemorySaver()
```

:::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";

const memory = new MemorySaver();
```

:::

这是一个内存中的检查点，在本教程中很方便。然而，在生产应用程序中，您可能会将其更改为使用 `SqliteSaver` 或 `PostgresSaver` 并连接数据库。

## 2. 编译图表

使用提供的检查点编译图表，这将作为图表处理每个节点时的 `State` 进行检查。

:::python

```python
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript hl_lines="7"
const graph = new StateGraph(State)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 3. 与您的聊天机器人互动

现在您可以与您的机器人互动了！

1.  选择一个要用作此对话键的线程。

    :::python

    ```python
    config = {"configurable": {"thread_id": "1"}}
    ```

    :::

    :::js

    ```typescript
    const config = { configurable: { thread_id: "1" } };
    ```

    :::

2.  调用您的聊天机器人：

    :::python

    ```python
    user_input = "Hi there! My name is Will."

    # config 是 stream() 或 invoke() 的 **第二个位置参数**！
    events = graph.stream(
        {"messages": [{"role": "user", "content": user_input}]},
        config,
        stream_mode="values",
    )
    for event in events:
        event["messages"][-1].pretty_print()
    ```

    ```
    ================================ Human Message =================================

    Hi there! My name is Will.
    ================================== Ai Message ==================================

    Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
    ```

    !!! note

        config 是在调用图表时作为**第二个位置参数**提供的。它没有嵌套在图表输入 (`{'messages': []}`) 中。

    :::

    :::js

    ```typescript
    const userInput = "Hi there! My name is Will.";

    const events = await graph.stream(
      { messages: [{ type: "human", content: userInput }] },
      { configurable: { thread_id: "1" }, streamMode: "values" }
    );

    for await (const event of events) {
      const lastMessage = event.messages.at(-1);
      console.log(`${lastMessage?.getType()}: ${lastMessage?.text}`);
    }
    ```

    ```
    human: Hi there! My name is Will.
    ai: Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?
    ```

    !!! note
    !!! note

        config 作为**第二个参数**提供给图表。它未嵌套在图表输入 (`{"messages": []}`) 中。

    :::

## 4. 提出后续问题

提出一个后续问题：

:::python

```python
user_input = "Remember my name?"

# config 是 stream() 或 invoke() 的 **第二个位置参数**！
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    config,
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.
```

:::

:::js

```typescript
const userInput2 = "Remember my name?";

const events2 = await graph.stream(
  { messages: [{ type: "human", content: userInput2 }] },
  { configurable: { thread_id: "1" }, streamMode: "values" }
);

for await (const event of events2) {
  const lastMessage = event.messages.at(-1);
  console.log(`${lastMessage?.getType()}: ${lastMessage?.text}`);
}
```

```
human: Remember my name?
ai: Yes, your name is Will. How can I help you today?
```

:::

请注意，我们没有使用外部列表进行记忆：这一切都由检查点处理！您可以在此 [LangSmith 跟踪](https://smith.langchain.com/public/29ba22b5-6d40-4fbe-8d27-b369e3329c84/r) 中检查完整的执行过程，以了解正在发生的情况。

不信？尝试使用不同的 config。

:::python

```python
# 唯一不同的是我们将 `thread_id` 从 "1" 改为 "2"
events = graph.stream(
    {"messages": [{"role": "user", "content": user_input}]},
    # highlight-next-line
    {"configurable": {"thread_id": "2"}},
    stream_mode="values",
)
for event in events:
    event["messages"][-1].pretty_print()
```

```
================================ Human Message =================================

Remember my name?
================================== Ai Message ==================================

I apologize, but I don't have any previous context or memory of your name. As an AI assistant, I don't retain information from past conversations. Each interaction starts fresh. Could you please tell me your name so I can address you properly in this conversation?
```

:::

:::js

```typescript hl_lines="3-4"
const events3 = await graph.stream(
  { messages: [{ type: "human", content: userInput2 }] },
  // 唯一不同的是我们将 `thread_id` 从 "1" 改为 "2"
  { configurable: { thread_id: "2" }, streamMode: "values" }
);

for await (const event of events3) {
  const lastMessage = event.messages.at(-1);
  console.log(`${lastMessage?.getType()}: ${lastMessage?.text}`);
}
```

```
human: Remember my name?
ai: I don't have the ability to remember personal information about users between interactions. However, I'm here to help you with any questions or topics you want to discuss!
```

:::

请注意，我们唯一的变化是修改了 config 中的 `thread_id`。请参阅此次调用的 [LangSmith 跟踪](https://smith.langchain.com/public/51a62351-2f0a-4058-91cc-9996c5561428/r) 以作比较。

## 5. 检查状态

:::python

到现在为止，我们在两个不同的线程中进行了几次检查点。但检查点包含什么？要随时检查给定 config 的图表的 `state`，请调用 `get_state(config)`。

```python
snapshot = graph.get_state(config)
snapshot
```

```
StateSnapshot(values={'messages': [HumanMessage(content='Hi there! My name is Will.', additional_kwargs={}, response_metadata={}, id='8c1ca919-c553-4ebf-95d4-b59a2d61e078'), AIMessage(content="Hello Will! It's nice to meet you. How can I assist you today? Is there anything specific you'd like to know or discuss?", additional_kwargs={}, response_metadata={'id': 'msg_01WTQebPhNwmMrmmWojJ9KXJ', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 405, 'output_tokens': 32}}, id='run-58587b77-8c82-41e6-8a90-d62c444a261d-0', usage_metadata={'input_tokens': 405, 'output_tokens': 32, 'total_tokens': 437}), HumanMessage(content='Remember my name?', additional_kwargs={}, response_metadata={}, id='daba7df6-ad75-4d6b-8057-745881cea1ca'), AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}, next=(), config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-93e0-6acc-8004-f2ac846575d2'}}, metadata={'source': 'loop', 'writes': {'chatbot': {'messages': [AIMessage(content="Of course, I remember your name, Will. I always try to pay attention to important details that users share with me. Is there anything else you'd like to talk about or any questions you have? I'm here to help with a wide range of topics or tasks.", additional_kwargs={}, response_metadata={'id': 'msg_01E41KitY74HpENRgXx94vag', 'model': 'claude-3-5-sonnet-20240620', 'stop_reason': 'end_turn', 'stop_sequence': None, 'usage': {'input_tokens': 444, 'output_tokens': 58}}, id='run-ffeaae5c-4d2d-4ddb-bd59-5d5cbf2a5af8-0', usage_metadata={'input_tokens': 444, 'output_tokens': 58, 'total_tokens': 502})]}}, 'step': 4, 'parents': {}}, created_at='2024-09-27T19:30:10.820758+00:00', parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef7d06e-859f-6206-8003-e1bd3c264b8f'}}, tasks=())
```

```python
snapshot.next  # (由于在本轮图中图表已结束，`next` 为空。如果您在图表调用过程中获取状态，next 会指示下一步将执行哪个节点)
```

:::

:::js

到现在为止，我们在两个不同的线程中进行了几次检查点。但检查点包含什么？要随时检查给定 config 的图表的 `state`，请调用 `getState(config)`。

```typescript
await graph.getState({ configurable: { thread_id: "1" } });
```

```typescript
{
  values: {
    messages: [
      HumanMessage {
        "id": "32fabcef-b3b8-481f-8bcb-fd83399a5f8d",
        "content": "Hi there! My name is Will.",
        "additional_kwargs": {},
        "response_metadata": {}
      },
      AIMessage {
        "id": "chatcmpl-BrPbTsCJbVqBvXWySlYoTJvM75Kv8",
        "content": "Hello Will! How can I assist you today?",
        "additional_kwargs": {},
        "response_metadata": {},
        "tool_calls": [],
        "invalid_tool_calls": []
      },
      HumanMessage {
        "id": "561c3aad-f8fc-4fac-94a6-54269a220856",
        "content": "Remember my name?",
        "additional_kwargs": {},
        "response_metadata": {}
      },
      AIMessage {
        "id": "chatcmpl-BrPbU4BhhsUikGbW37hYuF5vvnnE2",
        "content": "Yes, I remember your name, Will! How can I help you today?",
        "additional_kwargs": {},
        "response_metadata": {},
        "tool_calls": [],
        "invalid_tool_calls": []
      }
    ]
  },
  next: [],
  tasks: [],
  metadata: {
    source: 'loop',
    step: 4,
    parents: {},
    thread_id: '1'
  },
  config: {
    configurable: {
      thread_id: '1',
      checkpoint_id: '1f05cccc-9bb6-6270-8004-1d2108bcec77',
      checkpoint_ns: ''
    }
  },
  createdAt: '2025-07-09T13:58:27.607Z',
  parentConfig: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1f05cccc-78fa-68d0-8003-ffb01a76b599'
    }
  }
}
```

```typescript
import * as assert from "node:assert";

// 由于在本轮图中图表已结束，`next` 为空。
// 如果您在图表调用过程中获取状态，next 会指示下一步将执行哪个节点)
assert.deepEqual(snapshot.next, []);
```

:::

上面的快照包含当前状态值、相应的 config 以及要处理的 `next` 节点。在我们的例子中，图表已到达 `END` 状态，因此 `next` 为空。

**恭喜！** 您的聊天机器人现在可以通过 LangGraph 的检查点系统在会话中维护对话状态。这为更自然、更具上下文的交互打开了令人兴奋的可能性。LangGraph 的检查点甚至可以处理**任意复杂的图表状态**，这比简单的聊天记忆更具表现力和强大。

查看下面的代码片段，回顾本教程中的图表：

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python hl_lines="36 37"
from typing import Annotated

from langchain.chat_models import init_chat_model
from langchain_tavily import TavilySearch
from langchain_core.messages import BaseMessage
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph
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
graph_builder.add_edge("tools", "chatbot")
graph_builder.set_entry_point("chatbot")
memory = InMemorySaver()
graph = graph_builder.compile(checkpointer=memory)
```

:::

:::js

```typescript hl_lines="16 26"
import { END, MessagesZodState, START } from "@langchain/langgraph";
import { ChatOpenAI } from "@langchain/openai";
import { TavilySearch } from "@langchain/tavily";

import { MemorySaver } from "@langchain/langgraph";
import { StateGraph } from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { z } from "zod";

const State = z.object({
  messages: MessagesZodState.shape.messages,
});

const tools = [new TavilySearch({ maxResults: 2 })];
const llm = new ChatOpenAI({ model: "gpt-4o-mini" }).bindTools(tools);
const memory = new MemorySaver();

async function generateText(content: string) {

const graph = new StateGraph(State)
  .addNode("chatbot", async (state) => ({
    messages: [await llm.invoke(state.messages)],
  }))
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 后续步骤

在下一个教程中，您将[为聊天机器人添加人工干预](./4-human-in-the-loop.md)，以处理在继续之前可能需要指导或验证的情况。