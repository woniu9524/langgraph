# 如何将 LangGraph 集成到你的 React 应用中

!!! info "先决条件"

    - [LangGraph Platform](../../concepts/langgraph_platform.md)
    - [LangGraph Server](../../concepts/langgraph_server.md)

`useStream()` React hook 提供了一种无缝集成 LangGraph 到你的 React 应用中的方法。它处理了流式传输、状态管理和分支逻辑的所有复杂性，让你能够专注于构建出色的聊天体验。

主要特性：

- 消息流式传输：处理消息块的流，以形成一条完整消息
- 消息、中断、加载状态和错误的自动状态管理
- 对话分支：从聊天历史的任何点创建 वैकल्पिक 对话路径
- UI 无关设计：自带组件和样式

让我们探索一下如何在你的 React 应用中使用 `useStream()`。

`useStream()` 为创建定制化的聊天体验提供了坚实的基础。对于预先构建的聊天组件和界面，我们也推荐查看 [CopilotKit](https://docs.copilotkit.ai/coagents/quickstart/langgraph) 和 [assistant-ui](https://www.assistant-ui.com/docs/runtimes/langgraph)。

## 安装

```bash
npm install @langchain/langgraph-sdk @langchain/core
```

## 示例

```tsx
"use client";

import { useStream } from "@langchain/langgraph-sdk/react";
import type { Message } from "@langchain/langgraph-sdk";

export default function App() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [{ type: "human", content: message }] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            停止
          </button>
        ) : (
          <button key="submit" type="submit">发送</button>
        )}
      </form>
    </div>
  );
}
```

## 自定义你的 UI

`useStream()` hook 在后台处理所有复杂的无状态管理，为你提供简单的接口来构建你的 UI。以下是你可以开箱即用的功能：

- Thread 状态管理
- 加载和错误状态
- 中断
- 消息处理和更新
- 分支支持

以下是一些关于如何有效使用这些功能的示例：

### 加载状态

`isLoading` 属性会告诉你流是否处于活动状态，使你能够：

- 显示加载指示器
- 在处理过程中禁用输入字段
- 显示取消按钮

```tsx
export default function App() {
  const { isLoading, stop } = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <form>
      {isLoading && (
        <button key="stop" type="button" onClick={() => stop()}>
          停止
        </button>
      )}
    </form>
  );
}
```

### 页面刷新后恢复流

`useStream()` hook 可以通过设置 `reconnectOnMount: true` 来在挂载时自动恢复正在进行的运行。这对于在页面刷新后继续流式传输非常有用，确保在停机期间生成的任何消息和事件都不会丢失。

```tsx
const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: true,
});
```

默认情况下，创建的运行的 ID 存储在 `window.sessionStorage` 中，可以通过在 `reconnectOnMount` 中传递自定义存储来替换。该存储用于持久化线程的进行中运行 ID（在 `lg:stream:${threadId}` 键下）。

```tsx
const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  reconnectOnMount: () => window.localStorage,
});
```

你还可以通过使用运行回调来持久化运行元数据和 `joinStream` 函数来恢复流来手动管理恢复过程。确保在创建运行时传递 `streamResumable: true`；否则可能会丢失某些事件。

````tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useCallback, useState, useEffect, useRef } from "react";

export default function App() {
  const [threadId, onThreadId] = useSearchParam("threadId");

  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",

    threadId,
    onThreadId,

    onCreated: (run) => {
      window.sessionStorage.setItem(`resume:${run.thread_id}`, run.run_id);
    },
    onFinish: (_, run) => {
      window.sessionStorage.removeItem(`resume:${run?.thread_id}`);
    },
  });

  // 确保每个线程只加入一次流。
  const joinedThreadId = useRef<string | null>(null);
  useEffect(() => {
    if (!threadId) return;

    const resume = window.sessionStorage.getItem(`resume:${threadId}`);
    if (resume && joinedThreadId.current !== threadId) {
      thread.joinStream(resume);
      joinedThreadId.current = threadId;
    }
  }, [threadId]);

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const message = new FormData(form).get("message") as string;
        thread.submit(
          { messages: [{ type: "human", content: message }] },
          { streamResumable: true }
        );
      }}
    >
      <div>
        {thread.messages.map((message) => (
          <div key={message.id}>{message.content as string}</div>
        ))}
      </div>
      <input type="text" name="message" />
      <button type="submit">发送</button>
    </form>
  );
}

// 用于在 URL 中检索和持久化搜索参数的实用方法
function useSearchParam(key: string) {
  const [value, setValue] = useState<string | null>(() => {
    const params = new URLSearchParams(window.location.search);
    return params.get(key) ?? null;
  });

  const update = useCallback(
    (value: string | null) => {
      setValue(value);

      const url = new URL(window.location.href);
      if (value == null) {
        url.searchParams.delete(key);
      } else {
        url.searchParams.set(key, value);
      }

      window.history.pushState({}, "", url.toString());
    },
    [key]
  );

  return [value, update] as const;
}
````

### 线程管理

通过内置的线程管理来跟踪对话。你可以访问当前线程 ID，并在创建新线程时收到通知：

```tsx
const [threadId, setThreadId] = useState<string | null>(null);

const thread = useStream<{ messages: Message[] }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",

  threadId: threadId,
  onThreadId: setThreadId,
});
````

我们建议将 `threadId` 存储在 URL 的查询参数中，以便用户在刷新页面后可以恢复对话。

### 消息处理

`useStream()` hook 将跟踪从服务器接收到的消息块，并将它们连接起来形成一条完整消息。可以通过 `messages` 属性检索已完成的消息块。

默认情况下，`messagesKey` 设置为 `messages`，它会将新的消息块追加到 `values["messages"]`。如果将消息存储在不同的键中，则可以更改 `messagesKey` 的值。

```tsx
import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";

export default function HomePage() {
  const thread = useStream<{ messages: Message[] }>({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      {thread.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
}
```

在内部，`useStream()` hook 将使用 `streamMode: "messages-tuple"` 来接收来自你的图节点中任何 LangChain 聊天模型调用的消息流（即单独的 LLM token）。在 [streaming](../how-tos/streaming.md#messages) 指南中了解更多关于消息流式传输的信息。

### 中断

`useStream()` hook 暴露了 `interrupt` 属性，该属性将填充来自线程的最后一个中断。你可以使用中断来：

- 在执行节点之前渲染确认 UI
- 等待用户输入，允许代理向用户提出澄清性问题

在 [如何处理中断](../../how-tos/human_in_the_loop/wait-user-input.ipynb) 指南中了解更多关于中断的信息。

```tsx
const thread = useStream<{ messages: Message[] }, { InterruptType: string }>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

if (thread.interrupt) {
  return (
    <div>
      已中断！ {thread.interrupt.value}
      <button
        type="button"
        onClick={() => {
          // `resume` 可以是代理接受的任何值
          thread.submit(undefined, { command: { resume: true } });
        }}
      >
        恢复
      </button>
    </div>
  );
}
```

### 分支

对于每条消息，你可以使用 `getMessagesMetadata()` 来获取该消息首次出现时的第一个检查点。然后，你可以从第一个检查点之前的检查点创建一个新的运行，从而在线程中创建新的分支。

分支可以通过以下方式创建：

1. 编辑之前的用户消息。
2. 请求重新生成之前的助手消息。

```tsx
"use client";

import type { Message } from "@langchain/langgraph-sdk";
import { useStream } from "@langchain/langgraph-sdk/react";
import { useState } from "react";

function BranchSwitcher({
  branch,
  branchOptions,
  onSelect,
}: {
  branch: string | undefined;
  branchOptions: string[] | undefined;
  onSelect: (branch: string) => void;
}) {
  if (!branchOptions || !branch) return null;
  const index = branchOptions.indexOf(branch);

  return (
    <div className="flex items-center gap-2">
      <button
        type="button"
        onClick={() => {
          const prevBranch = branchOptions[index - 1];
          if (!prevBranch) return;
          onSelect(prevBranch);
        }}
      >
        上一个
      </button>
      <span>
        {index + 1} / {branchOptions.length}
      </span>
      <button
        type="button"
        onClick={() => {
          const nextBranch = branchOptions[index + 1];
          if (!nextBranch) return;
          onSelect(nextBranch);
        }}
      >
        下一个
      </button>
    </div>
  );
}

function EditMessage({
  message,
  onEdit,
}: {
  message: Message;
  onEdit: (message: Message) => void;
}) {
  const [editing, setEditing] = useState(false);

  if (!editing) {
    return (
      <button type="button" onClick={() => setEditing(true)}>
        编辑
      </button>
    );
  }

  return (
    <form
      onSubmit={(e) => {
        e.preventDefault();
        const form = e.target as HTMLFormElement;
        const content = new FormData(form).get("content") as string;

        form.reset();
        onEdit({ type: "human", content });
        setEditing(false);
      }}
    >
      <input name="content" defaultValue={message.content as string} />
      <button type="submit">保存</button>
    </form>
  );
}

export default function App() {
  const thread = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    messagesKey: "messages",
  });

  return (
    <div>
      <div>
        {thread.messages.map((message) => {
          const meta = thread.getMessagesMetadata(message);
          const parentCheckpoint = meta?.firstSeenState?.parent_checkpoint;

          return (
            <div key={message.id}>
              <div>{message.content as string}</div>

              {message.type === "human" && (
                <EditMessage
                  message={message}
                  onEdit={(message) =>
                    thread.submit(
                      { messages: [message] },
                      { checkpoint: parentCheckpoint }
                    )
                  }
                />
              )}

              {message.type === "ai" && (
                <button
                  type="button"
                  onClick={() =>
                    thread.submit(undefined, { checkpoint: parentCheckpoint })
                  }
                >
                  <span>重新生成</span>
                </button>
              )}

              <BranchSwitcher
                branch={meta?.branch}
                branchOptions={meta?.branchOptions}
                onSelect={(branch) => thread.setBranch(branch)}
              />
            </div>
          );
        })}
      </div>

      <form
        onSubmit={(e) => {
          e.preventDefault();

          const form = e.target as HTMLFormElement;
          const message = new FormData(form).get("message") as string;

          form.reset();
          thread.submit({ messages: [message] });
        }}
      >
        <input type="text" name="message" />

        {thread.isLoading ? (
          <button key="stop" type="button" onClick={() => thread.stop()}>
            停止
          </button>
        ) : (
          <button key="submit" type="submit">
            发送
          </button>
        )}
      </form>
    </div>
  );
}
```

对于高级用例，你可以使用 `experimental_branchTree` 属性来获取线程的树形表示，这可以用于渲染非消息型图的分支控件。

### 乐观更新

你可以在执行代理的网络请求之前乐观地更新客户端状态，这可以让你向用户提供即时反馈，例如在代理处理请求之前立即显示用户消息。

```tsx
const stream = useStream({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});

const handleSubmit = (text: string) => {
  const newMessage = { type: "human" as const, content: text };

  stream.submit(
    { messages: [newMessage] },
    {
      optimisticValues(prev) {
        const prevMessages = prev.messages ?? [];
        const newMessages = [...prevMessages, newMessage];
        return { ...prev, messages: newMessages };
      },
    }
  );
};
```

### 缓存线程显示

使用 `initialValues` 选项可立即显示缓存的线程数据，同时从服务器加载历史记录。这通过在导航到现有线程时立即显示缓存数据来改善用户体验。

```tsx
import { useStream } from "@langchain/langgraph-sdk/react";

const CachedThreadExample = ({ threadId, cachedThreadData }) => {
  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    // 在加载历史记录时立即显示缓存数据
    initialValues: cachedThreadData?.values,
    messagesKey: "messages",
  });

  return (
    <div>
      {stream.messages.map((message) => (
        <div key={message.id}>{message.content as string}</div>
      ))}
    </div>
  );
};
```

### 乐观线程创建

在 `submit` 函数中使用 `threadId` 选项来启用乐观 UI 模式，你需要在线程实际创建之前知道线程 ID。

```tsx
import { useState } from "react";
import { useStream } from "@langchain/langgraph-sdk/react";

const OptimisticThreadExample = () => {
  const [threadId, setThreadId] = useState<string | null>(null);
  const [optimisticThreadId] = useState(() => crypto.randomUUID());

  const stream = useStream({
    apiUrl: "http://localhost:2024",
    assistantId: "agent",
    threadId,
    onThreadId: setThreadId, // (3) 在线程创建后更新。
    messagesKey: "messages",
  });

  const handleSubmit = (text: string) => {
    // (1) 进行软导航到 /threads/${optimisticThreadId}
    // 无需等待线程创建。
    window.history.pushState({}, "", `/threads/${optimisticThreadId}`);

    // (2) 提交消息以使用预定 ID 创建线程。
    stream.submit(
      { messages: [{ type: "human", content: text }] },
      { threadId: optimisticThreadId }
    );
  };

  return (
    <div>
      <p>Thread ID: {threadId ?? optimisticThreadId}</p>
      {/* 组件的其余部分 */}
    </div>
  );
};
```

### TypeScript

`useStream()` hook 对使用 TypeScript 编写的应用非常友好，你可以为状态指定类型以获得更好的类型安全和 IDE 支持。

```tsx
// 定义你的类型
type State = {
  messages: Message[];
  context?: Record<string, unknown>;
};

// 在 hook 中使用它们
const thread = useStream<State>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

你还可以为不同场景选择性地指定类型，例如：

- `ConfigurableType`: `config.configurable` 属性的类型（默认为 `Record<string, unknown>`）
- `InterruptType`: 中断值的类型 - 即 `interrupt(...)` 函数的内容（默认为 `unknown`）
- `CustomEventType`: 自定义事件的类型（默认为 `unknown`）
- `UpdateType`: submit 函数的类型（默认为 `Partial<State>`）

```tsx
const thread = useStream<
  State,
  {
    UpdateType: {
      messages: Message[] | Message;
      context?: Record<string, unknown>;
    };
    InterruptType: string;
    CustomEventType: {
      type: "progress" | "debug";
      payload: unknown;
    };
    ConfigurableType: {
      model: string;
    };
  }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

如果你使用 LangGraph.js，你也可以重用你图的注解类型。但是，请确保仅导入注解模式的类型，以避免导入整个 LangGraph.js 运行时（例如，通过 `import type { ... }` 指令）。

```tsx
import {
  Annotation,
  MessagesAnnotation,
  type StateType,
  type UpdateType,
} from "@langchain/langgraph/web";

const AgentState = Annotation.Root({
  ...MessagesAnnotation.spec,
  context: Annotation<string>(),
});

const thread = useStream<
  StateType<typeof AgentState.spec>,
  { UpdateType: UpdateType<typeof AgentState.spec> }
>({
  apiUrl: "http://localhost:2024",
  assistantId: "agent",
  messagesKey: "messages",
});
```

## 事件处理

`useStream()` hook 提供了几个回调选项来帮助你响应不同事件：

- `onError`: 发生错误时调用。
- `onFinish`: 流结束时调用。
- `onUpdateEvent`: 收到更新事件时调用。
- `onCustomEvent`: 收到自定义事件时调用。请参阅 [streaming](../../how-tos/streaming.md#stream-custom-data) 指南了解如何流式传输自定义事件。
- `onMetadataEvent`: 收到元数据事件时调用，其中包含 Run ID 和 Thread ID。

## 了解更多

- [JS/TS SDK 参考](../reference/sdk/js_ts_sdk_ref.md)