# 使用 LangGraph 实现生成式用户界面

!!! info "前提条件"

    - [LangGraph Platform](../../concepts/langgraph_platform.md)
    - [LangGraph Server](../../concepts/langgraph_server.md)
    - [`useStream()` React Hook](./use_stream_react.md)

生成式用户界面（Generative UI）使代理能够超越文本，生成丰富的用户界面。这使得创建更具交互性和上下文感知能力的应用成为可能，其中用户界面可以根据对话流程和 AI 响应进行调整。

![Generative UI Sample](./img/generative_ui_sample.jpg)

LangGraph Platform 支持将您的 React 组件与图代码共置。这使您能够专注于为图构建特定的 UI 组件，同时轻松接入现有的聊天界面，如 [Agent Chat](https://agentchat.vercel.app)，并在真正需要时才加载代码。

## 教程

### 1. 定义和配置 UI 组件

首先，创建您的第一个 UI 组件。对于每个组件，您需要提供一个唯一的标识符，该标识符将用于在您的图代码中引用该组件。

```tsx title="src/agent/ui.tsx"
const WeatherComponent = (props: { city: string }) => {
  return <div>Weather for {props.city}</div>;
};

export default {
  weather: WeatherComponent,
};
```

接下来，在您的 `langgraph.json` 配置中定义您的 UI 组件：

```json
{
  "node_version": "20",
  "graphs": {
    "agent": "./src/agent/index.ts:graph"
  },
  "ui": {
    "agent": "./src/agent/ui.tsx"
  }
}
```

`ui` 部分指向将由图使用的 UI 组件。默认情况下，我们建议使用与图名称相同的键，但您可以根据需要拆分组件，有关详细信息请参见 [自定义 UI 组件的命名空间](#customise-the-namespace-of-ui-components)。

LangGraph Platform 将自动打包您的 UI 组件代码和样式，并将它们作为外部资源提供，这些资源可以被 `LoadExternalComponent` 组件加载。某些依赖项（如 `react` 和 `react-dom`）将自动从包中排除。

CSS 和 Tailwind 4.x 也开箱即用支持，因此您可以自由地在 UI 组件中使用 Tailwind 类以及 `shadcn/ui`。

=== "`src/agent/ui.tsx`"

    ```tsx
    import "./styles.css";

    const WeatherComponent = (props: { city: string }) => {
      return <div className="bg-red-500">Weather for {props.city}</div>;
    };

    export default {
      weather: WeatherComponent,
    };
    ```

=== "`src/agent/styles.css`"

    ```css
    @import "tailwindcss";
    ```

### 2. 在图表中发送 UI 组件

=== "Python"

    ```python title="src/agent.py"
    import uuid
    from typing import Annotated, Sequence, TypedDict

    from langchain_core.messages import AIMessage, BaseMessage
    from langchain_openai import ChatOpenAI
    from langgraph.graph import StateGraph
    from langgraph.graph.message import add_messages
    from langgraph.graph.ui import AnyUIMessage, ui_message_reducer, push_ui_message


    class AgentState(TypedDict):  # noqa: D101
        messages: Annotated[Sequence[BaseMessage], add_messages]
        ui: Annotated[Sequence[AnyUIMessage], ui_message_reducer]


    async def weather(state: AgentState):
        class WeatherOutput(TypedDict):
            city: str

        weather: WeatherOutput = (
            await ChatOpenAI(model="gpt-4o-mini")
            .with_structured_output(WeatherOutput)
            .with_config({"tags": ["nostream"]})
            .ainvoke(state["messages"])
        )

        message = AIMessage(
            id=str(uuid.uuid4()),
            content=f"Here's the weather for {weather['city']}",
        )

        # Emit UI elements associated with the message
        push_ui_message("weather", weather, message=message)
        return {"messages": [message]}


    workflow = StateGraph(AgentState)
    workflow.add_node(weather)
    workflow.add_edge("__start__", "weather")
    graph = workflow.compile()
    ```

=== "JS"

    使用 `typedUi` 工具从节点中发出 UI 元素：

    ```typescript title="src/agent/index.ts"
    import {
      typedUi,
      uiMessageReducer,
    } from "@langchain/langgraph-sdk/react-ui/server";

    import { ChatOpenAI } from "@langchain/openai";
    import { v4 as uuidv4 } from "uuid";
    import { z } from "zod";

    import type ComponentMap from "./ui.js";

    import {
      Annotation,
      MessagesAnnotation,
      StateGraph,
      type LangGraphRunnableConfig,
    } from "@langchain/langgraph";

    const AgentState = Annotation.Root({
      ...MessagesAnnotation.spec,
      ui: Annotation({ reducer: uiMessageReducer, default: () => [] }),
    });

    export const graph = new StateGraph(AgentState)
      .addNode("weather", async (state, config) => {
        // 提供组件映射的类型以确保
        // `ui.push()` 调用的类型安全，以及
        // 将消息推送到 `ui` 并同时发送自定义事件。
        const ui = typedUi<typeof ComponentMap>(config);

        const weather = await new ChatOpenAI({ model: "gpt-4o-mini" })
          .withStructuredOutput(z.object({ city: z.string() }))
          .withConfig({ tags: ["nostream"] })
          .invoke(state.messages);

        const response = {
          id: uuidv4(),
          type: "ai",
          content: `Here's the weather for ${weather.city}`,
        };

        // Emit UI elements associated with the AI message
        ui.push({ name: "weather", props: weather }, { message: response });

        return { messages: [response] };
      })
      .addEdge("__start__", "weather")
      .compile();
    ```

### 3. 在您的 React 应用程序中处理 UI 元素

在客户端，您可以使用 `useStream()` 和 `LoadExternalComponent` 来显示 UI 元素。

```tsx title="src/app/page.tsx"
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import { LoadExternalComponent } from "@langchain/langgraph-sdk/react-ui";

export default function Page() {
  const { thread, values } = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>
          {message.content}
          {values.ui
            ?.filter((ui) => ui.metadata?.message_id === message.id)
            .map((ui) => (
              <LoadExternalComponent key={ui.id} stream={thread} message={ui} />
            ))}
        </div>
      ))}
    </div>
  );
}
```

在后台，`LoadExternalComponent` 将从 LangGraph Platform 获取 UI 组件的 JS 和 CSS，并在 shadow DOM 中渲染它们，从而确保样式与您应用程序的其余部分隔离开来。

## 操作指南

### 在客户端提供自定义组件

如果您已经在客户端应用程序中加载了组件，您可以提供一个这些组件的映射，以便直接渲染，而无需从 LangGraph Platform 获取 UI 代码。

```tsx
const clientComponents = {
  weather: WeatherComponent,
};

<LoadExternalComponent
  stream={thread}
  message={ui}
  components={clientComponents}
/>;
```

### 在组件加载时显示加载 UI

您可以提供一个在组件加载时渲染的回退 UI。

```tsx
<LoadExternalComponent
  stream={thread}
  message={ui}
  fallback={<div>Loading...</div>}
/>
```

### 自定义 UI 组件的命名空间。

默认情况下，`LoadExternalComponent` 将使用 `useStream()` Hook 中的 `assistantId` 来获取 UI 组件的代码。您可以通过向 `LoadExternalComponent` 组件提供 `namespace` 属性来自定义此行为。

=== "`src/app/page.tsx`"

    ```tsx
    <LoadExternalComponent
      stream={thread}
      message={ui}
      namespace="custom-namespace"
    />
    ```

=== "`langgraph.json`"

    ```json
    {
      "ui": {
        "custom-namespace": "./src/agent/ui.tsx"
      }
    }
    ```

### 从 UI 组件访问和交互式线程状态

您可以使用 `useStreamContext` Hook 在 UI 组件中访问线程状态。

```tsx
import { useStreamContext } from "@langchain/langgraph-sdk/react-ui";

const WeatherComponent = (props: { city: string }) => {
  const { thread, submit } = useStreamContext();
  return (
    <>
      <div>Weather for {props.city}</div>

      <button
        onClick={() => {
          const newMessage = {
            type: "human",
            content: `What's the weather in ${props.city}?`,
          };

          submit({ messages: [newMessage] });
        }}
      >
        Retry
      </button>
    </>
  );
};
```

### 将附加上下文传递给客户端组件

您可以通过向 `LoadExternalComponent` 组件提供 `meta` 属性来将附加上下文传递给客户端组件。

```tsx
<LoadExternalComponent stream={thread} message={ui} meta={{ userId: "123" }} />
```

然后，您可以使用 `useStreamContext` Hook 在 UI 组件中访问 `meta` 属性。

```tsx
import { useStreamContext } from "@langchain/langgraph-sdk/react-ui";

const WeatherComponent = (props: { city: string }) => {
  const { meta } = useStreamContext<
    { city: string },
    { MetaType: { userId?: string } }
  >();

  return (
    <div>
      Weather for {props.city} (user: {meta?.userId})
    </div>
  );
};
```

### 从服务器流式传输 UI 消息

您可以通过使用 `useStream()` Hook 的 `onCustomEvent` 回调，在节点执行完成之前流式传输 UI 消息。当 LLM 生成响应时更新 UI 组件特别有用。

```tsx
import { uiMessageReducer } from "@langchain/langgraph-sdk/react-ui";

const { thread, submit } = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  onCustomEvent: (event, options) => {
    options.mutate((prev) => {
      const ui = uiMessageReducer(prev.ui ?? [], event);
      return { ...prev, ui };
    });
  },
});
```

然后，您可以通过调用 `ui.push()` / `push_ui_message()` 并使用您想要更新的 UI 消息相同的 ID，来将更新推送到 UI 组件。

=== "Python"

    ```python
    from typing import Annotated, Sequence, TypedDict

    from langchain_anthropic import ChatAnthropic
    from langchain_core.messages import AIMessage, AIMessageChunk, BaseMessage
    from langgraph.graph import StateGraph
    from langgraph.graph.message import add_messages
    from langgraph.graph.ui import AnyUIMessage, push_ui_message, ui_message_reducer


    class AgentState(TypedDict):  # noqa: D101
        messages: Annotated[Sequence[BaseMessage], add_messages]
        ui: Annotated[Sequence[AnyUIMessage], ui_message_reducer]


    class CreateTextDocument(TypedDict):
        """Prepare a document heading for the user."""

        title: str


    async def writer_node(state: AgentState):
        model = ChatAnthropic(model="claude-3-5-sonnet-latest")
        message: AIMessage = await model.bind_tools(
            tools=[CreateTextDocument],
            tool_choice={"type": "tool", "name": "CreateTextDocument"},
        ).ainvoke(state["messages"])

        tool_call = next(
            (x["args"] for x in message.tool_calls if x["name"] == "CreateTextDocument"),
            None,
        )

        if tool_call:
            ui_message = push_ui_message("writer", tool_call, message=message)
            ui_message_id = ui_message["id"]

            # We're already streaming the LLM response to the client through UI messages
            # so we don't need to stream it again to the `messages` stream mode.
            content_stream = model.with_config({"tags": ["nostream"]}).astream(
                f"Create a document with the title: {tool_call['title']}"
            )

            content: AIMessageChunk | None = None
            async for chunk in content_stream:
                content = content + chunk if content else chunk

                push_ui_message(
                    "writer",
                    {"content": content.text()},
                    id=ui_message_id,
                    message=message,
                    # Use `merge=rue` to merge props with the existing UI message
                    merge=True,
                )

        return {"messages": [message]}
    ```

=== "JS"

    ```tsx
    import {
      Annotation,
      MessagesAnnotation,
      type LangGraphRunnableConfig,
    } from "@langchain/langgraph";
    import { z } from "zod";
    import { ChatAnthropic } from "@langchain/anthropic";
    import {
      typedUi,
      uiMessageReducer,
    } from "@langchain/langgraph-sdk/react-ui/server";
    import type { AIMessageChunk } from "@langchain/core/messages";

    import type ComponentMap from "./ui";

    const AgentState = Annotation.Root({
      ...MessagesAnnotation.spec,
      ui: Annotation({ reducer: uiMessageReducer, default: () => [] }),
    });

    async function writerNode(
      state: typeof AgentState.State,
      config: LangGraphRunnableConfig
    ): Promise<typeof AgentState.Update> {
      const ui = typedUi<typeof ComponentMap>(config);

      const model = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });
      const message = await model
        .bindTools(
          [
            {
              name: "create_text_document",
              description: "Prepare a document heading for the user.",
              schema: z.object({ title: z.string() }),
            },
          ],
          { tool_choice: { type: "tool", name: "create_text_document" } }
        )
        .invoke(state.messages);

      type ToolCall = { name: "create_text_document"; args: { title: string } };
      const toolCall = message.tool_calls?.find(
        (tool): tool is ToolCall => tool.name === "create_text_document"
      );

      if (toolCall) {
        const { id, name } = ui.push(
          { name: "writer", props: { title: toolCall.args.title } },
          { message }
        );

        const contentStream = await model
          // We're already streaming the LLM response to the client through UI messages
          // so we don't need to stream it again to the `messages` stream mode.
          .withConfig({ tags: ["nostream"] })
          .stream(`Create a short poem with the topic: ${message.text}`);

        let content: AIMessageChunk | undefined;
        for await (const chunk of contentStream) {
          content = content?.concat(chunk) ?? chunk;

          ui.push(
            { id, name, props: { content: content?.text } },
            // Use `merge: true` to merge props with the existing UI message
            { message, merge: true }
          );
        }
      }

      return { messages: [message] };
    }
    ```

=== "`ui.tsx`"

    ```tsx
    function WriterComponent(props: { title: string; content?: string }) {
      return (
        <article>
          <h2>{props.title}</h2>
          <p style={{ whiteSpace: "pre-wrap" }}>{props.content}</p>
        </article>
      );
    }

    export default {
      weather: WriterComponent,
    };
    ```

### 从状态中移除 UI 消息

与可以通过追加 `RemoveMessage` 从状态中移除消息类似，您可以通过调用具有 UI 消息 ID 的 `remove_ui_message` / `ui.delete` 来从状态中移除 UI 消息。

=== "Python"

    ```python
    from langgraph.graph.ui import push_ui_message, delete_ui_message

    # push message
    message = push_ui_message("weather", {"city": "London"})

    # remove said message
    delete_ui_message(message["id"])
    ```

=== "JS"

    ```tsx
    // push message
    const message = ui.push({ name: "weather", props: { city: "London" } });

    // remove said message
    ui.delete(message.id);
    ```

## 了解更多

- [JS/TS SDK 参考](../reference/sdk/js_ts_sdk_ref.md)