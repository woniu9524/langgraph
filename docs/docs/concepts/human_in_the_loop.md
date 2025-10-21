---
search:
  boost: 2
tags:
  - human-in-the-loop
  - hil
  - overview
hide:
  - tags
---

# 人工介入 (Human-in-the-loop)

为了审查、编辑和批准代理或工作流中的工具调用，请使用 LangGraph 的人工介入功能（[human-in-the-loop features](../how-tos/human_in_the_loop/add-human-in-the-loop.md)），在工作流中的任何点启用人工干预。这对于大型语言模型 (LLM)-驱动的应用程序尤其有用，在这些应用程序中，模型输出可能需要验证、更正或附加上下文。

<figure markdown="1">
![image](../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
</figure>

!!! tip

    有关如何使用人工介入的信息，请参阅[启用人工干预](../how-tos/human_in_the_loop/add-human-in-the-loop.md)和[使用 Server API 进行人工介入](../cloud/how-tos/add-human-in-the-loop.md)。

## 主要功能

* **持久化执行状态**：中断利用 LangGraph 的[持久化](./persistence.md)层，该层可保存图状态，直到您恢复为止，无限期地暂停图执行。之所以能够实现这一点，是因为 LangGraph 在每个步骤后都会检查点（checkpoint）图状态，从而使系统能够保留执行上下文并稍后恢复工作流，从中断处继续。这支持异步的人工审查或输入，不受时间限制。

    有两种暂停图的方法：

    * [动态中断](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt)：使用 `interrupt` 命令，根据图的当前状态，从特定节点内部暂停图。
    * [静态中断](../how-tos/human_in_the_loop/add-human-in-the-loop.md#debug-with-interrupts)：使用 `interrupt_before` 和 `interrupt_after`，在预定义点（节点执行之前或之后）暂停图。

    <figure markdown="1">
    ![image](./img/breakpoints.png){: style="max-height:400px"}
    <figcaption>一个由 3 个顺序步骤组成的示例图，在 step_3 之前设置了断点。</figcaption> </figure>

* **灵活的集成点**：可以在工作流的任何点引入人工介入逻辑。这使得有针对性的人工参与成为可能，例如批准 API 调用、更正输出或指导对话。

## 模式

您可以使用 `interrupt` 和 `Command` 来实现四种典型的设计模式：

* [批准或拒绝](../how-tos/human_in_the_loop/add-human-in-the-loop.md#approve-or-reject)：在执行关键步骤（如 API 调用）之前暂停图，以审查和批准该操作。如果操作被拒绝，您可以阻止图执行该步骤，并可能采取替代措施。此模式通常涉及根据人工输入来路由图。
* [编辑图状态](../how-tos/human_in_the_loop/add-human_in_the_loop.md#review-and-edit-state)：暂停图以审查和编辑图状态。这对于更正错误或使用附加信息更新状态非常有用。此模式通常涉及使用人工输入来更新状态。
* [审查工具调用](../how-tos/human_in_the_loop/add-human_in_the_loop.md#review-tool-calls)：在工具执行之前，暂停图以审查和编辑 LLM 请求的工具调用。
* [验证人工输入](../how-tos/human_in_the_loop/add-human_in_the_loop.md#validate-human-input)：在进行下一步之前，暂停图以验证人工输入。