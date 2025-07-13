# 如何在同一线程上运行多个代理

在 LangGraph 平台中，线程并没有与特定的代理显式关联。
这意味着您可以在同一线程上运行多个代理，从而允许另一个代理从第一个代理的进度开始继续执行。

在本示例中，我们将创建两个代理，然后将它们都调用到同一个线程上。
您将看到第二个代理会利用第一个代理在线程中作为上下文生成的 [检查点](https://langchain-ai.github.io/langgraph/concepts/low_level/#checkpointer-state) 信息进行响应。

## 设置

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)

    openai_assistant = await client.assistants.create(
        graph_id="agent", config={"configurable": {"model_name": "openai"}}
    )

    # 应始终有一个不带配置的默认助手
    assistants = await client.assistants.search()
    default_assistant = [a for a in assistants if not a["config"]][0]
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    
    const openAIAssistant = await client.assistants.create(
      { graphId: "agent", config: {"configurable": {"model_name": "openai"}}}
    );

    const assistants = await client.assistants.search();
    const defaultAssistant = assistants.find(a => !a.config);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants \
        --header 'Content-Type: application/json' \
        --data '{
            "graph_id": "agent",
            "config": { "configurable": { "model_name": "openai" } }
        }' && \
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants/search \
        --header 'Content-Type: application/json' \
        --data '{
            "limit": 10,
            "offset": 0
        }' | jq -c 'map(select(.config == null or .config == {})) | .[0]'
    ```

我们可以看到这些代理是不同的：

=== "Python"

    ```python
    print(openai_assistant)
    ```

=== "Javascript"

    ```js
    console.log(openAIAssistant);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/assistants/<OPENAI_ASSISTANT_ID>
    ```

输出：

    {
        "assistant_id": "db87f39d-b2b1-4da8-ac65-cf81beb3c766",
        "graph_id": "agent",
        "created_at": "2024-08-30T21:18:51.850581+00:00",
        "updated_at": "2024-08-30T21:18:51.850581+00:00",
        "config": {
            "configurable": {
                "model_name": "openai"
            }
        },
        "metadata": {}
    }

=== "Python"

    ```python
    print(default_assistant)
    ```

=== "Javascript"

    ```js
    console.log(defaultAssistant);
    ```

=== "CURL"

    ```bash
    curl --request GET \
        --url <DEPLOYMENT_URL>/assistants/<DEFAULT_ASSISTANT_ID>
    ```

输出：

    {
        "assistant_id": "fe096781-5601-53d2-b2f6-0d3403f7e9ca",
        "graph_id": "agent",
        "created_at": "2024-08-08T22:45:24.562906+00:00",
        "updated_at": "2024-08-08T22:45:24.562906+00:00",
        "config": {},
        "metadata": {
            "created_by": "system"
        }
    }

## 在线程上运行代理

### 运行 OpenAI 助手

我们现在可以先在线程上运行 OpenAI 助手。

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        openai_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    const thread = await client.threads.create();
    let input =  {"messages": [{"role": "user", "content": "who made you?"}]}

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      openAIAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    thread_id=$(curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}' | jq -r '.thread_id') && \
    curl --request POST \
        --url "<DEPLOYMENT_URL>/threads/${thread_id}/runs/stream" \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <OPENAI_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "who made you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
        sed 's/\r$//' | \
        awk '
        /^event:/ {
            if (data_content != "") {
                print data_content "\n"
            }
            sub(/^event: /, "Receiving event of type: ", $0)
            printf "%s...\n", $0
            data_content = ""
        }
        /^data:/ {
            sub(/^data: /, "", $0)
            data_content = $0
        }
        END {
            if (data_content != "") {
                print data_content "\n\n"
            }
        }
    '
    ```

输出：

    Receiving event of type: metadata
    {'run_id': '1ef671c5-fb83-6e70-b698-44dba2d9213e'}


    Receiving event of type: updates
    {'agent': {'messages': [{'content': 'I was created by OpenAI, a research organization focused on developing and advancing artificial intelligence technology.', 'additional_kwargs': {}, 'response_metadata': {'finish_reason': 'stop', 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5'}, 'type': 'ai', 'name': None, 'id': 'run-f5735b86-b80d-4c71-8dc3-4782b5a9c7c8', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

### 运行默认助手

现在，我们可以在默认助手上运行它，可以看到第二个助手了解初始问题，并可以回答“你呢？”这个问题：

=== "Python"

    ```python
    input = {"messages": [{"role": "user", "content": "and you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        default_assistant["assistant_id"],
        input=input,
        stream_mode="updates",
    ):
        print(f"Receiving event of type: {event.event}")
        print(event.data)
        print("\n\n")
    ```

=== "Javascript"

    ```js
    let input =  {"messages": [{"role": "user", "content": "and you?"}]}

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      defaultAssistant["assistant_id"],
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const event of streamResponse) {
      console.log(`Receiving event of type: ${event.event}`);
      console.log(event.data);
      console.log("\n\n");
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/stream \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <DEFAULT_ASSISTANT_ID>,
            "input": {
                "messages": [
                    {
                        "role": "user",
                        "content": "and you?"
                    }
                ]
            },
            "stream_mode": [
                "updates"
            ]
        }' | \
        sed 's/\r$//' | \
        awk '
        /^event:/ {
            if (data_content != "") {
                print data_content "\n"
            }
            sub(/^event: /, "Receiving event of type: ", $0)
            printf "%s...\n", $0
            data_content = ""
        }
        /^data:/ {
            sub(/^data: /, "", $0)
            data_content = $0
        }
        END {
            if (data_content != "") {
                print data_content "\n\n"
            }
        }
    '
    ```

输出：

    Receiving event of type: metadata
    {'run_id': '1ef6722d-80b3-6fbb-9324-253796b1cd13'}


    Receiving event of type: updates
    {'agent': {'messages': [{'content': [{'text': 'I am an artificial intelligence created by Anthropic, not by OpenAI. I should not have stated that OpenAI created me, as that is incorrect. Anthropic is the company that developed and trained me using advanced language models and AI technology. I will be more careful about providing accurate information regarding my origins in the future.', 'type': 'text', 'index': 0}], 'additional_kwargs': {}, 'response_metadata': {'stop_reason': 'end_turn', 'stop_sequence': None}, 'type': 'ai', 'name': None, 'id': 'run-ebaacf62-9dd9-4165-9535-db432e4793ec', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': {'input_tokens': 302, 'output_tokens': 72, 'total_tokens': 374}}]}}