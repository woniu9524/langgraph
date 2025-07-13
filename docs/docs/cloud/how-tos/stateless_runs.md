# 无状态运行

大多数情况下，您会在运行图时向客户端提供 `thread_id`，以便通过 LangGraph Platform 中实现的持久化状态来跟踪之前的运行。然而，如果您不需要持久化运行，则无需使用内置的持久化状态，而是可以创建无状态运行。

## 设置

首先，我们来设置客户端：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    # 使用名为 "agent" 的已部署图
    assistant_id = "agent"
    # 创建 thread
    thread = await client.threads.create()
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    // 使用名为 "agent" 的已部署图
    const assistantId = "agent";
    // 创建 thread
    const thread = await client.threads.create();
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

## 无状态流式传输

我们可以以几乎与流式传输有状态运行相同的方式流式传输无状态运行的结果，但不是将值传递给 `thread_id` 参数，而是传递 `None`：

=== "Python"

    ```python
    input = {
        "messages": [
            {"role": "user", "content": "Hello! My name is Bagatur and I am 26 years old."}
        ]
    }

    async for chunk in client.runs.stream(
        # 不传入 thread_id，流将是无状态的
        None,
        assistant_id,
        input=input,
        stream_mode="updates",
    ):
        if chunk.data and "run_id" not in chunk.data:
            print(chunk.data)
    ```

=== "Javascript"

    ```js
    let input = {
      messages: [
        { role: "user", content: "Hello! My name is Bagatur and I am 26 years old." }
      ]
    };

    const streamResponse = client.runs.stream(
      // 不传入 thread_id，流将是无状态的
      null,
      assistantId,
      {
        input,
        streamMode: "updates"
      }
    );
    for await (const chunk of streamResponse) {
      if (chunk.data && !("run_id" in chunk.data)) {
        console.log(chunk.data);
      }
    }
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/stream \
        --header 'Content-Type: application/json' \
        --data "{
            \"assistant_id\": \"agent\",
            \"input\": {\"messages\": [{\"role\": \"human\", \"content\": \"Hello! My name is Bagatur and I am 26 years old.\"}]},
            \"stream_mode\": [
                \"updates\"
            ]
        }" | jq -c 'select(.data and (.data | has("run_id") | not)) | .data'
    ```

输出：

    {'agent': {'messages': [{'content': "Hello Bagatur! It's nice to meet you. Thank you for introducing yourself and sharing your age. Is there anything specific you'd like to know or discuss? I'm here to help with any questions or topics you're interested in.", 'additional_kwargs': {}, 'response_metadata': {}, 'type': 'ai', 'name': None, 'id': 'run-489ec573-1645-4ce2-a3b8-91b391d50a71', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}

## 等待无状态结果

除了流式传输之外，您还可以使用 `.wait` 函数来等待无状态结果，如下所示：

=== "Python"

    ```python
    stateless_run_result = await client.runs.wait(
        None,
        assistant_id,
        input=input,
    )
    print(stateless_run_result)
    ```

=== "Javascript"

    ```js
    let statelessRunResult = await client.runs.wait(
      null,
      assistantId,
      { input: input }
    );
    console.log(statelessRunResult);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/runs/wait \
        --header 'Content-Type: application/json' \
        --data '{
            "assistant_id": <ASSISTANT_IDD>,
        }'
    ```

输出：

    {
        'messages': [
            {
                'content': 'Hello! My name is Bagatur and I am 26 years old.',
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'human',
                'name': None,
                'id': '5e088543-62c2-43de-9d95-6086ad7f8b48',
                'example': False}
            ,
            {
                'content': "Hello Bagatur! It's nice to meet you. Thank you for introducing yourself and sharing your age. Is there anything specific you'd like to know or discuss? I'm here to help with any questions or topics you'd like to explore.",
                'additional_kwargs': {},
                'response_metadata': {},
                'type': 'ai',
                'name': None,
                'id': 'run-d6361e8d-4d4c-45bd-ba47-39520257f773',
                'example': False,
                'tool_calls': [],
                'invalid_tool_calls': [],
                'usage_metadata': None
            }
        ]
    }