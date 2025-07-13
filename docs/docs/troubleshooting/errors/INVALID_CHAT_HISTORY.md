# INVALID_CHAT_HISTORY

此错误是在预构建的 [create_react_agent][langgraph.prebuilt.chat_agent_executor.create_react_agent] 中，当 `call_model` 图节点接收到格式错误的消息列表时引发的。具体来说，当存在带有 `tool_calls`（LLM 请求调用工具）的 `AIMessages` 但没有对应的 `ToolMessage`（工具调用的结果，用于返回给 LLM）时，消息列表即被视为格式错误。

您看到此错误可能出于以下几个原因：

1. 您在调用图时手动传入了格式错误的消息列表，例如 `graph.invoke({'messages': [AIMessage(..., tool_calls=[...])]})`
2. 在收到 `tools` 节点（即 `ToolMessage` 列表）的更新之前，图就被中断了，并且您使用非 `None` 或 `ToolMessage` 的输入进行了调用，例如 `graph.invoke({'messages': [HumanMessage(...)]}, config)`。
    此中断可能由以下任一方式触发：
     - 您在 `create_react_agent` 中手动设置了 `interrupt_before = ['tools']`
     - `ToolNode` (`"tools"`) 未能处理其中一个工具引发的错误

## 故障排除

要解决此问题，您可以执行以下任一操作：

1. 不要使用格式错误的消息列表来调用图
2. 如果发生中断（手动或由于错误）：

    - 提供与现有工具调用匹配的 `ToolMessages`，然后调用 `graph.invoke({'messages': [ToolMessage(...)]})`。
    **注意**：这将把消息追加到历史记录中，并从 START 节点运行图。
    - 手动更新状态并从中恢复图：

        1. 使用 `graph.get_state(config)` 从图状态中获取最近消息的列表
        2. 修改消息列表，可以删除 `AIMessages` 中未回答的工具调用，或添加与未回答的工具调用匹配的 `ToolMessages`
        3. 使用修改后的消息列表调用 `graph.update_state(config, {'messages': ...})`
        4. 恢复图，例如调用 `graph.invoke(None, config)`