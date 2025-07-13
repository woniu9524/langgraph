---
search:
  boost: 2
---

# 重复发送消息

!!! info "先决条件"
    - [LangGraph Server](./langgraph_server.md)

很多时候用户可能会以非预期的方式与您的图进行交互。
例如，用户可能发送一条消息，在图完成运行之前又发送了第二条消息。
更普遍地说，用户可能在第一次运行完成之前调用图第二次。
我们将这种情况称为“重复发送消息”（double texting）。

目前，LangGraph 仅在 [LangGraph Platform](langgraph_platform.md) 中解决了此问题，开源版本未提供。
原因是，为了处理这个问题，我们需要了解图是如何部署的，由于 LangGraph Platform 负责部署，因此逻辑需要保留在那里。
如果您不想使用 LangGraph Platform，我们在下面详细介绍了我们已实现的所有选项。

![](img/double_texting.png)

## 拒绝 (Reject)

这是最简单的选项，它会拒绝任何后续运行，不允许重复发送消息。
请参阅 [操作指南](../cloud/how-tos/reject_concurrent.md) 配置拒绝重复发送消息的选项。

## 排队 (Enqueue)

这是一个相对简单的选项，它会继续执行第一次运行直到整个运行完成，然后将新输入作为一次单独的运行发送。
请参阅 [操作指南](../cloud/how-tos/enqueue_concurrent.md) 配置排队重复发送消息的选项。

## 中断 (Interrupt)

此选项会中断当前执行，但会保存到目前为止所做的所有工作。
然后它会插入用户输入并从那里继续。

如果启用此选项，您的图应能够处理可能出现的奇怪的边缘情况。
例如，您可能调用了一个工具，但尚未收到该工具运行的结果。
您可能需要删除该工具调用，以免留下悬空的工具调用。

请参阅 [操作指南](../cloud/how-tos/interrupt_concurrent.md) 配置中断重复发送消息的选项。

## 回滚 (Rollback)

此选项会中断当前执行，并回滚到目前为止所做的所有工作（包括原始运行输入）。然后它会插入新的用户输入，基本就像原始输入一样。

请参阅 [操作指南](../cloud/how-tos/rollback_concurrent.md) 配置回滚重复发送消息的选项。