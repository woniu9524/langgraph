# 自定义状态

在本教程中，您将为状态添加额外的字段，以定义复杂的行为，而无需依赖消息列表。聊天机器人将使用其搜索工具查找特定信息，并将其转发给人工审核。

!!! note

    本教程建立在 [添加人工审核控制](./4-human-in-the-loop.md) 的基础上。

## 1. 向状态添加键

通过向状态添加 `name` 和 `birthday` 键，更新聊天机器人以研究实体的生日：

:::python

```python
from typing import Annotated

from typing_extensions import TypedDict

from langgraph.graph.message import add_messages


class State(TypedDict):
    messages: Annotated[list, add_messages]
    # highlight-next-line
    name: str
    # highlight-next-line
    birthday: str
```

:::

:::js

```typescript
import { MessagesZodState } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({
  messages: MessagesZodState.shape.messages,
  // highlight-next-line
  name: z.string(),
  // highlight-next-line
  birthday: z.string(),
});
```

:::

将此信息添加到状态中，可以使其被其他图节点（如存储或处理信息的下游节点）以及图的持久化层轻松访问。

## 2. 在工具中更新状态

:::python

现在，在 `human_assistance` 工具中填充状态键。这允许人工在信息存储到状态之前对其进行审核。使用 [`Command`](../../concepts/low_level.md#using-inside-tools) 从工具内部发出状态更新。

```python
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool

from langgraph.types import Command, interrupt

@tool
# 请注意，由于我们要为状态更新生成 ToolMessage，
# 通常需要相应工具调用的 ID。我们可以使用
# LangChain 的 InjectedToolCallId 来指示不应向模型显示此参数
# 在工具的 schema 中。
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """请求人工协助。"""
    human_response = interrupt(
        {
            "question": "是否正确？",
            "name": name,
            "birthday": birthday,
        },
    )
    # 如果信息正确，则按原样更新状态。
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    # 否则，接收人工审阅者的信息。
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"已更正： {human_response}"

    # 这次我们显式地使用工具内部的 ToolMessage 来更新状态。
    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }
    # 我们将 Command 对象返回到工具中以更新我们的状态。
    return Command(update=state_update)
```

:::

:::js

现在，在 `humanAssistance` 工具中填充状态键。这允许用户在信息存储到状态之前对其进行审核。使用 [`Command`](../../concepts/low_level.md#using-inside-tools) 从工具内部发出状态更新。

```typescript
import { tool } from "@langchain/core/tools";
import { ToolMessage } from "@langchain/core/messages";
import { Command, interrupt } from "@langchain/langgraph";

const humanAssistance = tool(
  async (input, config) => {
    // 注意，因为我们需要为状态更新生成 ToolMessage，
    // 通常需要相应工具调用的 ID。
    // 该 ID 在工具的 config 中可用。
    const toolCallId = config?.toolCall?.id as string | undefined;
    if (!toolCallId) throw new Error("Tool call ID is required");

    const humanResponse = await interrupt({
      question: "Is this correct?",
      name: input.name,
      birthday: input.birthday,
    });

    // 我们显式地使用工具内部的 ToolMessage 来更新状态。
    const stateUpdate = (() => {
      // 如果信息正确，则按原样更新状态。
      if (humanResponse.correct?.toLowerCase().startsWith("y")) {
        return {
          name: input.name,
          birthday: input.birthday,
          messages: [
            new ToolMessage({ content: "Correct", tool_call_id: toolCallId }),
          ],
        };
      }

      // 否则，接收人工审阅者的信息。
      return {
        name: humanResponse.name || input.name,
        birthday: humanResponse.birthday || input.birthday,
        messages: [
          new ToolMessage({
            content: `Made a correction: ${JSON.stringify(humanResponse)}`,
            tool_call_id: toolCallId,
          }),
        ],
      };
    })();

    // 我们将 Command 对象返回到工具中以更新我们的状态。
    return new Command({ update: stateUpdate });
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      name: z.string().describe("The name of the entity"),
      birthday: z.string().describe("The birthday/release date of the entity"),
    }),
  }
);
```

:::

图的其余部分保持不变。

## 3. 提示聊天机器人

:::python
提示聊天机器人查找 LangGraph 库的“生日”，并指示聊天机器人在获取所需信息后调用 `human_assistance` 工具。通过在工具的参数中设置 `name` 和 `birthday`，可以强制聊天机器人生成这些字段的建议。

```python
user_input = (
    "你能查一下 LangGraph 是什么时候发布的吗？ "
    "拿到答案后，请使用 human_assistance 工具进行审核。"
)
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

:::

:::js
提示聊天机器人查找 LangGraph 库的“生日”，并指示聊天机器人在获取所需信息后调用 `humanAssistance` 工具。通过在工具的参数中设置 `name` 和 `birthday`，可以强制聊天机器人生成这些字段的建议。

```typescript
import { isAIMessage } from "@langchain/core/messages";

const userInput =
  "Can you look up when LangGraph was released? " +
  "When you have the answer, use the humanAssistance tool for review.";

const events = await graph.stream(
  { messages: [{ role: "user", content: userInput }] },
  { configurable: { thread_id: "1" }, streamMode: "values" }
);

for await (const event of events) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);

    console.log(
      "=".repeat(32),
      `${lastMessage?.getType()} Message`,
      "=".repeat(32)
    );
    console.log(lastMessage?.text);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
      console.log("Tool Calls:");
      for (const call of lastMessage.tool_calls) {
        console.log(`  ${call.name} (${call.id})`);
        console.log(`  Args: ${JSON.stringify(call.args)}`);
      }
    }
  }
}
```

:::

```
================================ Human Message =================================

你能查一下 LangGraph 是什么时候发布的吗？ 拿到答案后，请使用 human_assistance 工具进行审核。
================================== Ai Message ==================================

[{'text': "当然！我将首先使用 Tavily 搜索功能查找有关 LangGraph 发布日期信息。然后，我将使用 human_assistance 工具进行审核。", 'type': 'text'}, {'id': 'toolu_01JoXQPgTVJXiuma8xMVwqAi', 'input': {'query': 'LangGraph release date'}, 'name': 'tavily_search_results_json', 'type': 'tool_use'}]
Tool Calls:
  tavily_search_results_json (toolu_01JoXQPgTVJXiuma8xMVwqAi)
 Call ID: toolu_01JoXQPgTVJXiuma8xMVwqAi
  Args:
    query: LangGraph release date
================================= Tool Message =================================
Name: tavily_search_results_json

[{"url": "https://blog.langchain.dev/langgraph-cloud/", "content": "We also have a new stable release of LangGraph. By LangChain 6 min read Jun 27, 2024 (Oct '24) Edit: Since the launch of LangGraph Platform, we now have multiple deployment options alongside LangGraph Studio - which now fall under LangGraph Platform. LangGraph Platform is synonymous with our Cloud SaaS deployment option."}, {"url": "https://changelog.langchain.com/announcements/langgraph-cloud-deploy-at-scale-monitor-carefully-iterate-boldly", "content": "LangChain - Changelog | ☁ 🚀 LangGraph Platform: Deploy at scale, monitor LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain LangSmith LangGraph LangChain Changelog Sign up for our newsletter to stay up to date DATE: The LangChain Team LangGraph LangGraph Platform ☁ 🚀 LangGraph Platform: Deploy at scale, monitor carefully, iterate boldly DATE: June 27, 2024 AUTHOR: The LangChain Team LangGraph Platform is now in closed beta, offering scalable, fault-tolerant deployment for LangGraph agents. LangGraph Platform also includes a new playground-like studio for debugging agent failure modes and quick iteration: Join the waitlist today for LangGraph Platform. And to learn more, read our blog post announcement or check out our docs. Subscribe By clicking subscribe, you accept our privacy policy and terms and conditions."}]
================================== Ai Message ==================================

[{'text': "根据搜索结果，LangGraph 在 2024 年 6 月 27 日 LangGraph 平台发布之前就已经存在。然而，搜索结果并未提供 LangGraph 的具体发布日期。\n\n鉴于这些信息，我将使用 human_assistance 工具来审核并可能提供关于 LangGraph 初始发布日期的更准确信息。", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
```

:::python
我们又一次触发了 `human_assistance` 工具中的 `interrupt`。
:::

:::js
我们又一次触发了 `humanAssistance` 工具中的 `interrupt`。
:::

## 4. 添加人工协助

聊天机器人未能识别正确的日期，请为其提供信息：

:::python

```python
human_command = Command(
    resume={
        "name": "LangGraph",
        "birthday": "Jan 17, 2024",
    },
)

events = graph.stream(human_command, config, stream_mode="values")
for event in events:
    if "messages" in event:
        event["messages"][-1].pretty_print()
```

:::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const humanCommand = new Command({
  resume: {
    name: "LangGraph",
    birthday: "Jan 17, 2024",
  },
});

const resumeEvents = await graph.stream(humanCommand, {
  configurable: { thread_id: "1" },
  streamMode: "values",
});

for await (const event of resumeEvents) {
  if ("messages" in event) {
    const lastMessage = event.messages.at(-1);

    console.log(
      "=".repeat(32),
      `${lastMessage?.getType()} Message`,
      "=".repeat(32)
    );
    console.log(lastMessage?.text);

    if (
      lastMessage &&
      isAIMessage(lastMessage) &&
      lastMessage.tool_calls?.length
    ) {
      console.log("Tool Calls:");
      for (const call of lastMessage.tool_calls) {
        console.log(`  ${call.name} (${call.id})`);
        console.log(`  Args: ${JSON.stringify(call.args)}`);
      }
    }
  }
}
```

:::

```
================================== Ai Message ==================================

[{'text': "根据搜索结果，LangGraph 在 2024 年 6 月 27 日 LangGraph 平台发布之前就已经存在。然而，搜索结果并未提供 LangGraph 的具体发布日期。\n\n鉴于这些信息，我将使用 human_assistance 工具来审核并可能提供关于 LangGraph 初始发布日期的更准确信息。", 'type': 'text'}, {'id': 'toolu_01JDQAV7nPqMkHHhNs3j3XoN', 'input': {'name': 'Assistant', 'birthday': '2023-01-01'}, 'name': 'human_assistance', 'type': 'tool_use'}]
Tool Calls:
  human_assistance (toolu_01JDQAV7nPqMkHHhNs3j3XoN)
 Call ID: toolu_01JDQAV7nPqMkHHhNs3j3XoN
  Args:
    name: Assistant
    birthday: 2023-01-01
================================= Tool Message =================================
Name: human_assistance

Made a correction: {'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
================================== Ai Message ==================================

感谢您的人工协助。我现在可以向您提供 LangGraph 发布日期的正确信息。

LangGraph 最初发布于 2024 年 1 月 17 日。此信息来自人工协助更正，比我最初找到的搜索结果更准确。

总结如下：
1. LangGraph 的原始发布日期：2024 年 1 月 17 日
2. LangGraph 平台公告：2024 年 6 月 27 日

值得注意的是，LangGraph 在 LangGraph 平台公告之前就已经开发和使用了一段时间，但 LangGraph 本身的官方初始发布日期是 2024 年 1 月 17 日。
```

现在，这些字段已反映在状态中：

:::python

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```
{'name': 'LangGraph', 'birthday': 'Jan 17, 2024'}
```

:::

:::js

```typescript
const snapshot = await graph.getState(config);

const relevantState = Object.fromEntries(
  Object.entries(snapshot.values).filter(([k]) =>
    ["name", "birthday"].includes(k)
  )
);
```

```
{ name: 'LangGraph', birthday: 'Jan 17, 2024' }
```

:::

这使得它们可以被下游节点（例如，进一步处理或存储信息的节点）轻松访问。

## 5. 手动更新状态

:::python
LangGraph 提供了对应用程序状态的高度控制。例如，在任何时候（包括中断时），您都可以使用 `graph.update_state` 手动覆盖一个键：

```python
graph.update_state(config, {"name": "LangGraph (library)"})
```

```
{'configurable': {'thread_id': '1',
  'checkpoint_ns': '',
  'checkpoint_id': '1efd4ec5-cf69-6352-8006-9278f1730163'}}
```

:::

:::js
LangGraph 提供了对应用程序状态的高度控制。例如，在任何时候（包括中断时），您都可以使用 `graph.updateState` 手动覆盖一个键：

```typescript
await graph.updateState(
  { configurable: { thread_id: "1" } },
  { name: "LangGraph (library)" }
);
```

```typescript
{
  configurable: {
    thread_id: '1',
    checkpoint_ns: '',
    checkpoint_id: '1efd4ec5-cf69-6352-8006-9278f1730163'
  }
}
```

:::

## 6. 查看新值

:::python
如果您调用 `graph.get_state`，您会看到新值已反映出来：

```python
snapshot = graph.get_state(config)

{k: v for k, v in snapshot.values.items() if k in ("name", "birthday")}
```

```
{'name': 'LangGraph (library)', 'birthday': 'Jan 17, 2024'}
```

:::

:::js
如果您调用 `graph.getState`，您会看到新值已反映出来：

```typescript
const updatedSnapshot = await graph.getState(config);

const updatedRelevantState = Object.fromEntries(
  Object.entries(updatedSnapshot.values).filter(([k]) =>
    ["name", "birthday"].includes(k)
  )
);
```

```typescript
{ name: 'LangGraph (library)', birthday: 'Jan 17, 2024' }
```

:::

手动状态更新将 [生成 LangSmith 的追踪](https://smith.langchain.com/public/7ebb7827-378d-49fe-9f6c-5df0e90086c8/r)。如果需要，它们也可用于 [控制人工审核工作流](../../how-tos/human_in_the_loop/add-human-in-the-loop.md)。通常建议使用 `interrupt` 函数，因为它允许数据在人工审核交互中独立于状态更新进行传输。

**恭喜！** 您已向状态添加了自定义键以促进更复杂的 Wflow，并学会了如何从工具内部生成状态更新。

请查看下面的代码片段，回顾本教程中的图：

:::python

{% include-markdown "../../../snippets/chat_model_tabs.md" %}

<!---
```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("anthropic:claude-3-5-sonnet-latest")
```
-->

```python
from typing import Annotated

from langchain_tavily import TavilySearch
from langchain_core.messages import ToolMessage
from langchain_core.tools import InjectedToolCallId, tool
from typing_extensions import TypedDict

from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph, START, END
from langgraph.graph.message import add_messages
from langgraph.prebuilt import ToolNode, tools_condition
from langgraph.types import Command, interrupt

class State(TypedDict):
    messages: Annotated[list, add_messages]
    name: str
    birthday: str

@tool
def human_assistance(
    name: str, birthday: str, tool_call_id: Annotated[str, InjectedToolCallId]
) -> str:
    """请求人工协助。"""
    human_response = interrupt(
        {
            "question": "是否正确？",
            "name": name,
            "birthday": birthday,
        },
    )
    if human_response.get("correct", "").lower().startswith("y"):
        verified_name = name
        verified_birthday = birthday
        response = "Correct"
    else:
        verified_name = human_response.get("name", name)
        verified_birthday = human_response.get("birthday", birthday)
        response = f"已更正： {human_response}"

    state_update = {
        "name": verified_name,
        "birthday": verified_birthday,
        "messages": [ToolMessage(response, tool_call_id=tool_call_id)],
    }
    return Command(update=state_update)


tool = TavilySearch(max_results=2)
tools = [tool, human_assistance]
llm_with_tools = llm.bind_tools(tools)

def chatbot(state: State):
    message = llm_with_tools.invoke(state["messages"])
    assert(len(message.tool_calls) <= 1) # LangGraph API changed, tool_calls is now a property of the message
    return {"messages": [message]}

graph_builder = StateGraph(State)
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
  Command,
  interrupt,
  MessagesZodState,
  MemorySaver,
  StateGraph,
  END,
  START,
} from "@langchain/langgraph";
import { ToolNode, toolsCondition } from "@langchain/langgraph/prebuilt";
import { ChatAnthropic } from "@langchain/anthropic";
import { TavilySearch } from "@langchain/tavily";
import { ToolMessage } from "@langchain/core/messages";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

const State = z.object({
  messages: MessagesZodState.shape.messages,
  name: z.string(),
  birthday: z.string(),
});

const humanAssistance = tool(
  async (input, config) => {
    // Note that because we are generating a ToolMessage for a state update, we
    // generally require the ID of the corresponding tool call. This is available
    // in the tool's config.
    const toolCallId = config?.toolCall?.id as string | undefined;
    if (!toolCallId) throw new Error("Tool call ID is required");

    const humanResponse = await interrupt({
      question: "Is this correct?",
      name: input.name,
      birthday: input.birthday,
    });

    // We explicitly update the state with a ToolMessage inside the tool.
    const stateUpdate = (() => {
      // If the information is correct, update the state as-is.
      if (humanResponse.correct?.toLowerCase().startsWith("y")) {
        return {
          name: input.name,
          birthday: input.birthday,
          messages: [
            new ToolMessage({ content: "Correct", tool_call_id: toolCallId }),
          ],
        };
      }

      // Otherwise, receive information from the human reviewer.
      return {
        name: humanResponse.name || input.name,
        birthday: humanResponse.birthday || input.birthday,
        messages: [
          new ToolMessage({
            content: `Made a correction: ${JSON.stringify(humanResponse)}`,
            tool_call_id: toolCallId,
          }),
        ],
      };
    })();

    // We return a Command object in the tool to update our state.
    return new Command({ update: stateUpdate });
  },
  {
    name: "humanAssistance",
    description: "Request assistance from a human.",
    schema: z.object({
      name: z.string().describe("The name of the entity"),
      birthday: z.string().describe("The birthday/release date of the entity"),
    }),
  }
);

const searchTool = new TavilySearch({ maxResults: 2 });

const tools = [searchTool, humanAssistance];
const llmWithTools = new ChatAnthropic({
  model: "claude-3-5-sonnet-latest",
}).bindTools(tools);

const memory = new MemorySaver();

const chatbot = async (state: z.infer<typeof State>) => {
  const message = await llmWithTools.invoke(state.messages);
  return { messages: message };
};

const graph = new StateGraph(State)
  .addNode("chatbot", chatbot)
  .addNode("tools", new ToolNode(tools))
  .addConditionalEdges("chatbot", toolsCondition, ["tools", END])
  .addEdge("tools", "chatbot")
  .addEdge(START, "chatbot")
  .compile({ checkpointer: memory });
```

:::

## 下一步

在完成 LangGraph 基础教程之前，还有最后一个概念需要回顾：将 `checkpointing` 和 `state updates` 与 [时间旅行](./6-time-travel.md) 连接起来。