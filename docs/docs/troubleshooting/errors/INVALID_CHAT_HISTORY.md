# 历史记录无效

:::python
当预构建的 @[create_react_agent][create_react_agent] 中的 `call_model` 图节点接收到格式错误的 mensajes 列表时，会抛出此错误。具体来说，当存在带有 `tool_calls`（LLM 请求调用工具）的 `AIMessages`，但没有对应的 `ToolMessage`（将工具调用结果返回给 LLM）时，该列表就被视为格式错误。
:::

:::js
当预构建的 @[createReactAgent][createReactAgent] 中的 `callModel` 图节点接收到格式错误的 messages 列表时，您会看到此错误。具体来说，当存在带有 `tool_calls`（LLM 要求调用工具）的 `AIMessage`，但没有对应的 `ToolMessage`（工具调用结果返回给 LLM）时，该列表就是格式错误的。
:::

您看到此错误可能有几个原因：

:::python

1. 在调用图时手动传递了格式错误的 messages 列表，例如 `graph.invoke({'messages': [AIMessage(..., tool_calls=[...])]})`
2. 图在收到 `tools` 节点（即 `ToolMessage` 列表）的更新之前被中断，并且您以非 None 或非 `ToolMessage` 的输入调用了它，例如 `graph.invoke({'messages': [HumanMessage(...)]}, config)`。

   此中断可能由以下任一方式触发：

   - 在 `create_react_agent` 中手动设置了 `interrupt_before = ['tools']`
   - `ToolNode`（“tools”）未处理的某个工具引发了错误

:::

:::js

1. 在调用图时手动传递了格式错误的 messages 列表，例如 `graph.invoke({messages: [new AIMessage({..., tool_calls: [...]})]})`
2. 图在收到 `tools` 节点（即 `ToolMessage` 列表）更新之前被中断，并且您使用非 null 或非 `ToolMessage` 的输入调用了它，例如 `graph.invoke({messages: [new HumanMessage(...)]}, config)`。

   此中断可能由以下任一方式触发：

   - 在 `createReactAgent` 中手动设置了 `interruptBefore: ['tools']`
   - `ToolNode`（“tools”）未处理的某个工具引发了错误

:::

## 疑难解答

要解决此问题，您可以执行以下任一操作：

1. 不要使用格式错误的 messages 列表调用图
2. 在中断（手动或由于错误）的情况下，您可以：

:::python

- 提供与现有工具调用匹配的 `ToolMessage`，然后调用 `graph.invoke({'messages': [ToolMessage(...)]})`。
  **注意**：这将把 messages 追加到历史记录中，并从 START 节点运行图。

  - 手动更新状态并从中断处恢复图：

          1. 使用 `graph.get_state(config)` 从图状态中获取最近的 messages 列表
          2. 修改 messages 列表，以移除 `AIMessages` 中未回答的工具调用，

          或者添加与未回答的工具调用匹配的 `ToolMessage` 3. 使用修改后的 messages 列表调用 `graph.update_state(config, {'messages': ...})` 4. 恢复图，例如调用 `graph.invoke(None, config)`
:::

:::js

- 提供与现有工具调用匹配的 `ToolMessage`，然后调用 `graph.invoke({messages: [new ToolMessage(...)]})`。
  **注意**：这将把 messages 追加到历史记录中，并从 START 节点运行图。

  - 手动更新状态并从中断处恢复图：

          1. 使用 `graph.getState(config)` 从图状态中获取最近的 messages 列表
          2. 修改 messages 列表，以移除 `AIMessage` 中未回答的工具调用，

          或者添加与未回答的工具调用匹配的 `toolCallId` 的 `ToolMessage` 3. 使用修改后的 messages 列表调用 `graph.updateState(config, {messages: ...})` 4. 恢复图，例如调用 `graph.invoke(null, config)`
:::