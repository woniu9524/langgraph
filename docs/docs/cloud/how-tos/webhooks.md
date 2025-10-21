# 使用 Webhook

在使用 LangGraph 平台时，您可能希望在 API 调用完成后接收更新，这时可以使用 Webhook。Webhook 非常适合在 run 处理完毕后触发您的服务中的操作。要实现此功能，您需要公开一个可以接受 `POST` 请求的端点，并将此端点作为 `webhook` 参数传递给您的 API 请求。

目前，SDK 不提供定义 Webhook 端点的内置支持，但您可以通过 API 请求手动指定。

## 支持的端点

以下 API 端点接受 `webhook` 参数：

| 操作         | HTTP 方法 | 端点                            |
| ------------ | --------- | ------------------------------- |
| Create Run   | `POST`    | `/thread/{thread_id}/runs`      |
| Create Thread Cron | `POST`    | `/thread/{thread_id}/runs/crons` |
| Stream Run   | `POST`    | `/thread/{thread_id}/runs/stream` |
| Wait Run     | `POST`    | `/thread/{thread_id}/runs/wait`  |
| Create Cron  | `POST`    | `/runs.crons`                   |
| Stream Run Stateless | `POST`    | `/runs/stream`                  |
| Wait Run Stateless | `POST`    | `/runs/wait`                    |

在本指南中，我们将展示如何在使用 Stream Run 后触发 Webhook。

## 设置您的 Assistant 和 Thread

在进行 API 调用之前，请设置您的 Assistant 和 Thread。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    assistant_id = "agent"
    thread = await client.threads.create()
    print(thread)
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const assistantID = "agent";
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{ "limit": 10, "offset": 0 }' | jq -c 'map(select(.config == null or .config == {})) | .[0]' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

示例响应：

```json
{
    "thread_id": "9dde5490-2b67-47c8-aa14-4bfec88af217",
    "created_at": "2024-08-30T23:07:38.242730+00:00",
    "updated_at": "2024-08-30T23:07:38.242730+00:00",
    "metadata": {},
    "status": "idle",
    "config": {},
    "values": null
}
```

## 将 Webhook 与 Graph Run 结合使用

要使用 Webhook，请在您的 API 请求中指定 `webhook` 参数。当 run 完成时，LangGraph 平台会向指定的 Webhook URL 发送一个 `POST` 请求。

例如，如果您的服务器监听的 Webhook 事件端点是 `https://my-server.app/my-webhook-endpoint`，请将其包含在您的请求中：

=== "Python"

    ```python
    input = { "messages": [{ "role": "user", "content": "Hello!" }] }

    async for chunk in client.runs.stream(
        thread_id=thread["thread_id"],
        assistant_id=assistant_id,
        input=input,
        stream_mode="events",
        webhook="https://my-server.app/my-webhook-endpoint"
    ):
        pass
    ```

=== "JavaScript"

    ```js
    const input = { messages: [{ role: "human", content: "Hello!" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      assistantID,
      {
        input: input,
        webhook: "https://my-server.app/my-webhook-endpoint"
      }
    );

    for await (const chunk of streamResponse) {
      // Handle stream output
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
            "input": {"messages": [{"role": "user", "content": "Hello!"}]},
            "webhook": "https://my-server.app/my-webhook-endpoint"
        }'
    ```

## Webhook Payload

LangGraph 平台以 [Run](../../concepts/assistants.md#execution) 的格式发送 Webhook 通知。有关详细信息，请参阅 [API 参考](https://langchain-ai.github.io/langgraph/cloud/reference/api/api_ref.html#model/run)。请求负载在 `kwargs` 字段中包含 run 的输入、配置和其他元数据。

## 安全 Webhook

为确保只有授权请求才能到达您的 Webhook 端点，请考虑在查询参数中添加安全令牌：

```
https://my-server.app/my-webhook-endpoint?token=YOUR_SECRET_TOKEN
```

您的服务器应在处理请求之前提取并验证此令牌。

## 禁用 Webhook

自 `langgraph-api>=0.2.78` 起，开发者可以在 `langgraph.json` 文件中禁用 Webhook：

```json
{
  "http": {
    "disable_webhooks": true
  }
}
```

此功能主要用于自托管部署，平台管理员或开发者可能更倾向于禁用 Webhook 以简化其安全配置——尤其是在不配置防火墙规则或其他网络控件的情况下。禁用 Webhook 有助于防止将不受信任的负载发送到内部端点。

有关完整的配置详细信息，请参阅 [配置文件参考](https://langchain-ai.github.io/langgraph/cloud/reference/cli/?h=disable_webhooks#configuration-file)。

## 测试 Webhook

您可以使用在线服务测试您的 Webhook，例如：

- **[Beeceptor](https://beeceptor.com/)** – 快速创建测试端点并检查传入的 Webhook 负载。
- **[Webhook.site](https://webhook.site/)** – 实时查看、调试和记录传入的 Webhook 请求。

这些工具可帮助您验证 LangGraph 平台是否正确触发并向您的服务发送 Webhook。