# 使用线程

在本指南中，我们将展示如何创建、查看和检查 [线程](../../concepts/persistence.md#threads)。

## 创建线程

要运行图和持久化状态，您必须先创建一个线程。

### 空线程

要创建新线程，请使用 [LangGraph SDK](../../concepts/sdk.md) 的 `create` 方法。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.create) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create_3) SDK 参考文档。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    thread = await client.threads.create()

    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const thread = await client.threads.create();

    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
    ```

Output:

    {
      "thread_id": "123e4567-e89b-12d3-a456-426614174000",
      "created_at": "2025-05-12T14:04:08.268Z",
      "updated_at": "2025-05-12T14:04:08.268Z",
      "metadata": {},
      "status": "idle",
      "values": {}
    }

### 复制线程

或者，如果您在应用程序中已有想要复制状态的线程，则可以使用 `copy` 方法。这将创建一个独立线程，其历史记录在操作时与原始线程相同。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.copy) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#copy) SDK 参考文档。

=== "Python"

    ```python
    copied_thread = await client.threads.copy(<THREAD_ID>)
    ```

=== "Javascript"

    ```js
    const copiedThread = await client.threads.copy(<THREAD_ID>);
    ```

=== "CURL"

    ```bash
    curl --request POST --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/copy \
    --header 'Content-Type: application/json'
    ```

### 预填充状态

最后，您可以通过向 `create` 方法提供 `supersteps` 列表来创建具有任意预定义状态的线程。`supersteps` 描述了状态更新序列的列表。例如：

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    thread = await client.threads.create(
      graph_id="agent",
      supersteps=[
        {
          updates: [
            {
              values: {},
              as_node: '__input__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    type: 'human',
                    content: 'hello',
                  },
                ],
              },
              as_node: '__start__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    content: 'Hello! How can I assist you today?',
                    type: 'ai',
                  },
                ],
              },
              as_node: 'call_model',
            },
          ],
        },
      ])

    print(thread)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const thread = await client.threads.create({
        graphId: 'agent',
        supersteps: [
        {
          updates: [
            {
              values: {},
              asNode: '__input__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    type: 'human',
                    content: 'hello',
                  },
                ],
              },
              asNode: '__start__',
            },
          ],
        },
        {
          updates: [
            {
              values: {
                messages: [
                  {
                    content: 'Hello! How can I assist you today?',
                    type: 'ai',
                  },
                ],
              },
              asNode: 'call_model',
            },
          ],
        },
      ],
    });

    console.log(thread);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{"metadata":{"graph_id":"agent"},"supersteps":[{"updates":[{"values":{},"as_node":"__input__"}]},{"updates":[{"values":{"messages":[{"type":"human","content":"hello"}]},"as_node":"__start__"}]},{"updates":[{"values":{"messages":[{"content":"Hello\u0021 How can I assist you today?","type":"ai"}]},"as_node":"call_model"}]}]}'
    ```

Output:

    {
        "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
        "created_at": "2025-05-12T15:37:08.935038+00:00",
        "updated_at": "2025-05-12T15:37:08.935046+00:00",
        "metadata": {"graph_id": "agent"},
        "status": "idle",
        "config": {},
        "values": {
            "messages": [
                {
                    "content": "hello",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "8701f3be-959c-4b7c-852f-c2160699b4ab",
                    "example": false
                },
                {
                    "content": "Hello! How can I assist you today?",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "ai",
                    "name": null,
                    "id": "4d8ea561-7ca1-409a-99f7-6b67af3e1aa3",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": null
                }
            ]
        }
    }

## 列出线程

### LangGraph SDK

要列出线程，请使用 [LangGraph SDK](../../concepts/sdk.md) 的 `search` 方法。这将列出应用程序中与提供的过滤器匹配的线程。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.ThreadsClient.search) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#search_2) SDK 参考文档。

#### 按线程状态过滤

使用 `status` 字段根据线程状态过滤线程。支持的值为 `idle`、`busy`、`interrupted` 和 `error`。有关每种状态的信息，请参阅 [此处](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/?h=thread+status#langgraph_sdk.auth.types.ThreadStatus)。例如，要查看 `idle` 线程：

=== "Python"

    ```python
    print(await client.threads.search(status="idle",limit=1))
    ```

=== "Javascript"

    ```js
    console.log(await client.threads.search({ status: "idle", limit: 1 }));
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"status": "idle", "limit": 1}'
    ```

Output:

    [
      {
        'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
        'created_at': '2024-08-14T17:36:38.921660+00:00',
        'updated_at': '2024-08-14T17:36:38.921660+00:00',
        'metadata': {'graph_id': 'agent'},
        'status': 'idle',
        'config': {'configurable': {}}
      }
    ]

#### 按元数据过滤

`search` 方法允许您按元数据进行过滤：

=== "Python"

    ```python
    print((await client.threads.search(metadata={"graph_id":"agent"},limit=1)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.search({ metadata: { "graph_id": "agent" }, limit: 1 })));
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/search \
    --header 'Content-Type: application/json' \
    --data '{"metadata": {"graph_id":"agent"}, "limit": 1}'
    ```

Output:

    [
      {
        'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
        'created_at': '2024-08-14T17:36:38.921660+00:00',
        'updated_at': '2024-08-14T17:36:38.921660+00:00',
        'metadata': {'graph_id': 'agent'},
        'status': 'idle',
        'config': {'configurable': {}}
      }
    ]

#### 排序

SDK 还支持使用 `sort_by` 和 `sort_order` 参数按 `thread_id`、`status`、`created_at` 和 `updated_at` 对线程进行排序。

### LangGraph Platform UI

您也可以通过 LangGraph Platform UI 在部署中查看线程。

在您的部署中，选择 "Threads"（线程）选项卡。这将加载您部署中所有线程的表格。

要按线程状态过滤，请选择顶部栏中的状态。要按支持的属性排序，请单击所需列的箭头图标。

## 检查线程

### LangGraph SDK

#### 获取线程

要通过 `thread_id` 查看特定线程，请使用 `get` 方法：

=== "Python"

    ```python
    print((await client.threads.get(<THREAD_ID>)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.get(<THREAD_ID>)));
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID> \
    --header 'Content-Type: application/json'
    ```

Output:

    {
      'thread_id': 'cacf79bb-4248-4d01-aabc-938dbd60ed2c',
      'created_at': '2024-08-14T17:36:38.921660+00:00',
      'updated_at': '2024-08-14T17:36:38.921660+00:00',
      'metadata': {'graph_id': 'agent'},
      'status': 'idle',
      'config': {'configurable': {}}
    }

#### 检查线程状态

要查看给定线程的当前状态，请使用 `get_state` 方法：

=== "Python"

    ```python
    print((await client.threads.get_state(<THREAD_ID>)))
    ```

=== "Javascript"

    ```js
    console.log((await client.threads.getState(<THREAD_ID>)));
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
    --header 'Content-Type: application/json'
    ```

Output:

    {
        "values": {
            "messages": [
                {
                    "content": "hello",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "human",
                    "name": null,
                    "id": "8701f3be-959c-4b7c-852f-c2160699b4ab",
                    "example": false
                },
                {
                    "content": "Hello! How can I assist you today?",
                    "additional_kwargs": {},
                    "response_metadata": {},
                    "type": "ai",
                    "name": null,
                    "id": "4d8ea561-7ca1-409a-99f7-6b67af3e1aa3",
                    "example": false,
                    "tool_calls": [],
                    "invalid_tool_calls": [],
                    "usage_metadata": null
                }
            ]
        },
        "next": [],
        "tasks": [],
        "metadata": {
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955",
            "graph_id": "agent_with_quite_a_long_name",
            "source": "update",
            "step": 1,
            "writes": {
                "call_model": {
                    "messages": [
                        {
                            "content": "Hello! How can I assist you today?",
                            "type": "ai"
                        }
                    ]
                }
            },
            "parents": {}
        },
        "created_at": "2025-05-12T15:37:09.008055+00:00",
        "checkpoint": {
            "checkpoint_id": "1f02f46f-733f-6b58-8001-ea90dcabb1bd",
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_ns": ""
        },
        "parent_checkpoint": {
            "checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955",
            "thread_id": "f15d70a1-27d4-4793-a897-de5609920b7d",
            "checkpoint_ns": ""
        },
        "checkpoint_id": "1f02f46f-733f-6b58-8001-ea90dcabb1bd",
        "parent_checkpoint_id": "1f02f46f-7308-616c-8000-1b158a9a6955"
    }

选择性地，要查看给定检查点的线程状态，只需传入检查点 ID（或整个检查点对象）：

=== "Python"

    ```python
    thread_state = await client.threads.get_state(
      thread_id=<THREAD_ID>
      checkpoint_id=<CHECKPOINT_ID>
    )
    ```

=== "Javascript"

    ```js
    const threadState = await client.threads.getState(<THREAD_ID>, <CHECKPOINT_ID>);
    ```

=== "CURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state/<CHECKPOINT_ID> \
    --header 'Content-Type: application/json'
    ```

#### 检查完整线程历史

要查看线程的历史记录，请使用 `get_history` 方法。这将返回线程经历的每个状态的列表。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/?h=thread+status#langgraph_sdk.client.ThreadsClient.get_history) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#gethistory) 参考文档。

### LangGraph Platform UI

您也可以通过 LangGraph Platform UI 在部署中查看线程。

在您的部署中，选择 "Threads"（线程）选项卡。这将加载您部署中所有线程的表格。

选择一个线程以检查其当前状态。要查看其完整历史记录并进一步进行调试，请在 [LangGraph Studio](../../concepts//langgraph_studio.md) 中打开该线程。