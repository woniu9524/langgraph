---
search:
  boost: 2
tags:
  - 人工介入回路
  - hil
  - 概述
hide:
  - tags
---

# 人工介入回路

要审查、编辑和批准代理或工作流中的工具调用，请使用 LangGraph 的[人工介入回路功能](../how-tos/human_in_the_loop/add-human-in-the-loop.md) 在工作流中的任何点启用人工干预。这对于大型语言模型 (LLM) 驱动的应用程序特别有用，在这些应用程序中，模型输出可能需要验证、更正或提供额外上下文。

<figure markdown="1">
![image](../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
</figure>

!!! tip

    有关如何使用人工介入回路的信息，请参阅[启用人工干预](../how-tos/human_in_the_loop/add-human_in_the_loop.md)和[使用 Server API 进行人工介入](../cloud/how-tos/add-human-in-the-loop.md)。

## 关键功能

* **持久的执行状态**: 您的工具调用可以通过 LangGraph 的[持久性](../../concepts/persistence.md)层进行中断，该层会保存图的状态，从而无限期地暂停图执行，直到您恢复。这是可能的，因为 LangGraph 在每个步骤后都会检查图的状态，这允许系统保留执行上下文并稍后恢复工作流，从中断处继续。这支持异步人工审查或无时间限制的输入。

    有<strong>两种</strong>暂停图的方法：

    - [动态中断](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt)：使用 `interrupt` 从特定节点内部暂停图，基于图的当前状态。
    - [静态中断](../how-tos/human_in_the_loop/add-human-in-the-loop.md#debug-with-interrupts)：使用 `interrupt_before` 和 `interrupt_after` 在定义好的点暂停图，可以在节点执行之前或之后。

    <figure markdown="1">
    ![image](./img/breakpoints.png){: style="max-height:400px"}
    <figcaption>一个由 3 个顺序步骤组成的示例图，在 step_3 之前有一个断点。</figcaption> </figure>

* **灵活的集成点**: 人工介入回路逻辑可以在工作流的任何点引入。这允许有针对性的人工参与，例如批准 API 调用、更正输出或指导对话。

## 模式

您可以使用 `interrupt` 和 `Command` 实现四种典型设计模式：

- [批准或拒绝](../how-tos/human_in_the_loop/add-human-in-the-loop.md#approve-or-reject)：在关键步骤（如 API 调用）之前暂停图，以审查和批准该操作。如果操作被拒绝，您可以阻止图执行该步骤，并可能采取替代措施。此模式通常涉及基于人工输入的路由图。
- [编辑图状态](../how-tos/human_in_the_loop/add-human-in-the-loop.md#review-and-edit-state)：暂停图以为审查和编辑图状态。这对于更正错误或使用附加信息更新状态很有用。此模式通常涉及使用人工输入更新状态。
- [审查工具调用](../how-tos/human_in_the_loop/add-human-in-the-loop.md#review-tool-calls)：在工具执行之前，暂停图以为 LLM 请求的工具调用进行审查和编辑。
- [验证人工输入](../how-tos/human_in_the_loop/add-human-in-the-loop.md#validate-human-input)：在继续下一步之前，暂停图以为人工输入进行验证。