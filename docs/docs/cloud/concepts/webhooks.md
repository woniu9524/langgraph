# Webhooks

Webhooks 使您的 LangGraph Platform 应用能够与外部服务进行事件驱动的通信。例如，您可能希望在 LangGraph Platform 的 API 调用运行完成后，向另一个服务发出更新通知。

许多 LangGraph Platform 端点接受 `webhook` 参数。如果一个可以接受 POST 请求的端点指定了此参数，LangGraph Platform 将在运行完成后发送一个请求。

有关更多详细信息，请参阅相应的 [操作指南](../../cloud/how-tos/webhooks.md)。