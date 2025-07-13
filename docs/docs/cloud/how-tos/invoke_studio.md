# 运行应用程序

!!!info  "先决条件"
    - [运行 Agents](../../agents/run_agents.md#running-agents)

本指南展示了如何向您的应用程序提交一次[运行](../../concepts/assistants.md#execution)。

## Graph 模式

### 指定输入
首先在页面左侧的“Input”部分（位于 Graph 界面下方）定义 Graph 的输入。

Studio 会尝试根据 Graph 定义的[状态 schema](../../concepts/low_level.md/#schema) 来渲染一个输入表单。若要禁用此功能，请点击“View Raw”按钮，它会弹出一个 JSON 编辑器。

点击“Input”部分顶部的向上/向下箭头，可以切换并使用之前提交过的输入。

### 运行设置

#### Assistant
要指定用于运行的[Assistant](../../concepts/assistants.md)，请点击左下角的设置按钮。如果当前已选择一个 Assistant，按钮上也会显示其名称。如果未选择 Assistant，则显示“Manage Assistants”。

选择要运行的 Assistant，然后点击模态框顶部的“Active”切换按钮来激活它。[此处](./studio/manage_assistants.md)有更多关于管理 Assistant 的信息。

#### Streaming
点击“Submit”旁边的下拉菜单，然后点击切换按钮来启用/禁用流式输出。

#### Breakpoints
若要使用断点运行您的 Graph，请点击“Interrupt”按钮。选择一个节点以及是在该节点执行之前或之后暂停（或两者都暂停）。点击线程日志中的“Continue”以恢复执行。

有关断点的更多信息，请[此处](../../concepts/human_in_the_loop.md)。

### 提交运行

要使用指定的输入和运行设置提交运行，请点击“Submit”按钮。这将向当前选定的[线程](../../concepts/persistence.md#threads)添加一次[运行](../../concepts/assistants.md#execution)。如果当前未选择任何线程，则会创建一个新的线程。

要取消正在进行的运行，请点击“Cancel”按钮。


## Chat 模式
在对话面板底部指定您的聊天应用程序的输入。点击“Send message”按钮将输入作为 Human 消息提交，并将响应流式传输回来。

要取消正在进行的运行，请点击“Cancel”按钮。点击“Show tool calls”切换按钮可以隐藏/显示对话中的工具调用。

## 了解更多

要从现有线程中的特定检查点运行您的应用程序，请参阅[本指南](./threads_studio.md#edit-thread-history)。