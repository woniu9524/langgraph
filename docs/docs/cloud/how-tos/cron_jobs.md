# 使用 Cron 作业

有时你可能不希望根据用户交互来运行图表，而是希望安排图表按计划运行——例如，如果你希望你的图表能够生成并发送每周待办事项邮件给你的团队。LangGraph Platform 允许你通过使用 `Crons` 客户端来完成此操作，而无需编写自己的脚本。要安排图表作业，你需要传递一个 [cron 表达式](https://crontab.cronhub.io/) 来告知客户端你希望何时运行图表。`Cron` 作业在后台运行，不会干扰图表的正常调用。

## 设置

首先，让我们设置我们的 SDK 客户端、助手和线程：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为 "agent" 的已部署图表
    assistant_id = "agent"
    # 创建线程
    thread = await client.threads.create()
    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为 "agent" 的已部署图表
    const assistantId = "agent";
    // 创建线程
    const thread = await client.threads.create();
    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0].graph_id' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

输出：

    {
        'thread_id': '9dde5490-2b67-47c8-aa14-4bfec88af217',
        'created_at': '2024-08-30T23:07:38.242730+00:00',
        'updated_at': '2024-08-30T23:07:38.242730+00:00',
        'metadata': {},
        'status': 'idle',
        'config': {},
        'values': None
    }

## 线程上的 Cron 作业

要创建与特定线程关联的 cron 作业，你可以编写：

=== "Python"

    ```python
    # 这将安排一个作业每天在 15:27（下午 3:27）运行
    cron_job = await client.crons.create_for_thread(
        thread["thread_id"],
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // 这将安排一个作业每天在 15:27（下午 3:27）运行
    const cronJob = await client.crons.create_for_thread(
      thread["thread_id"],
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

请注意，删除不再有用的 `Cron` 作业**非常**重要。否则，你可能会产生不必要的 LLM API 费用！你可以使用以下代码删除一个 `Cron` 作业：

=== "Python"

    ```python
    await client.crons.delete(cron_job["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJob["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```

## 无状态 Cron 作业

你也可以通过使用以下代码创建无状态 cron 作业：

=== "Python"

    ```python
    # 这将安排一个作业每天在 15:27（下午 3:27）运行
    cron_job_stateless = await client.crons.create(
        assistant_id,
        schedule="27 15 * * *",
        input={"messages": [{"role": "user", "content": "What time is it?"}]},
    )
    ```

=== "Javascript"

    ```js
    // 这将安排一个作业每天在 15:27（下午 3:27）运行
    const cronJobStateless = await client.crons.create(
      assistantId,
      {
        schedule: "27 15 * * *",
        input: { messages: [{ role: "user", content: "What time is it?" }] }
      }
    );
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/crons \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_ID>,
        }'
    ```

同样，请记住在完成后删除你的作业！

=== "Python"

    ```python
    await client.crons.delete(cron_job_stateless["cron_id"])
    ```

=== "Javascript"

    ```js
    await client.crons.delete(cronJobStateless["cron_id"]);
    ```

=== "CURL"

    ```bash
    curl --request DELETE \
        --url <DEPLOYMENT_URL>/runs/crons/<CRON_ID>
    ```