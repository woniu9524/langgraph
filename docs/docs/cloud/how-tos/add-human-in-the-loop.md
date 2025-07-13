# 人工干预：使用 Server API

要审核、编辑和批准代理或工作流中的工具调用，请使用 LangGraph 的 [人工干预](../../concepts/human_in_the_loop.md) 功能。

## 动态中断

=== "Python"

    ```python
    from langgraph_sdk import get_client
    # highlight-next-line
    from langgraph_sdk.schema import Command
    client = get_client(url=<DEPLOYMENT_URL>)

    # 使用名为 "agent" 的已部署图表
    assistant_id = "agent"

    # 创建一个线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 运行图表直到命中中断点。
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={"some_text": "original text"}   # (1)!
    )

    print(result['__interrupt__']) # (2)!
    # > [
    # >     {
    # >         'value': {'text_to_revise': 'original text'},
    # >         'resumable': True,
    # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    # >         'when': 'during'
    # >     }
    # > ]


    # 恢复图表
    print(await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        command=Command(resume="Edited text")   # (3)!
    ))
    # > {'some_text': 'Edited text'}
    ```

    1. 图表以初始状态调用。
    2. 当图表命中中断点时，它会返回一个包含有效负载和元数据的中断对象。
    3. 图表使用 `Command(resume=...)` 恢复，注入人工输入并继续执行。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // 使用名为 "agent" 的已部署图表
    const assistantID = "agent";

    // 创建一个线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 运行图表直到命中中断点。
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: { "some_text": "original text" } }   # (1)!
    );

    console.log(result['__interrupt__']); // (2)!
    // > [
    # >     {
    # >         'value': {'text_to_revise': 'original text'},
    # >         'resumable': True,
    # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    # >         'when': 'during'
    # >     }
    # > ]

    // 恢复图表
    console.log(await client.runs.wait(
        threadID,
        assistantID,
        // highlight-next-line
        { command: { resume: "Edited text" }}   # (3)!
    ));
    # > {'some_text': 'Edited text'}
    ```

    1. 图表以初始状态调用。
    2. 当图表命中中断点时，它会返回一个包含有效负载和元数据的中断对象。
    3. 图表使用 `{ resume: ... }` 命令对象恢复，注入人工输入并继续执行。

=== "cURL"

    创建线程：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    运行图表直到命中中断点：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"some_text\": \"original text\"}
    }"
    ```

    恢复图表：

    ```bash
    curl --request POST \
     --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
     --header 'Content-Type: application/json' \
     --data "{
       \"assistant_id\": \"agent\",
       \"command\": {
         \"resume\": \"Edited text\"
       }
     }"
    ```

??? example "扩展示例：使用 `interrupt`"

    这是您可以在 LangGraph API 服务器中运行的示例图表。
    有关更多详细信息，请参阅 [LangGraph Platform 快速入门](../quick_start.md)。

    ```python
    from typing import TypedDict
    import uuid

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.constants import START
    from langgraph.graph import StateGraph
    # highlight-next-line
    from langgraph.types import interrupt, Command

    class State(TypedDict):
        some_text: str

    def human_node(state: State):
        # highlight-next-line
        value = interrupt( # (1)!
            {
                "text_to_revise": state["some_text"] # (2)!
            }
        )
        return {
            "some_text": value # (3)!
        }


    # 构建图表
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")

    graph = graph_builder.compile()
    ```

    1. `interrupt(...)` 在 `human_node` 处暂停执行，将给定有效负载呈现给人工。
    2. 可以将任何 JSON 可序列化值传递给 `interrupt` 函数。此处为一个包含要修改的文本的字典。
    3. 一旦恢复，`interrupt(...)` 的返回值就是用户提供的输入，用于更新状态。

    一旦您运行了 LangGraph API 服务器，就可以使用 [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 进行交互。

    === "Python"

        ```python
        from langgraph_sdk import get_client
        # highlight-next-line
        from langgraph_sdk.schema import Command
        client = get_client(url=<DEPLOYMENT_URL>)

        # 使用名为 "agent" 的已部署图表
        assistant_id = "agent"

        # 创建一个线程
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # 运行图表直到命中中断点。
        result = await client.runs.wait(
            thread_id,
            assistant_id,
            input={"some_text": "original text"}   # (1)!
        )

        print(result['__interrupt__']) # (2)!
        # > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'resumable': True,
        # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        # >         'when': 'during'
        # >     }
        # > ]


        # 恢复图表
        print(await client.runs.wait(
            thread_id,
            assistant_id,
            # highlight-next-line
            command=Command(resume="Edited text")   # (3)!
        ))
        # > {'some_text': 'Edited text'}
        ```

        1. 图表以初始状态调用。
        2. 当图表命中中断点时，它会返回一个包含有效负载和元数据的中断对象。
        3. 图表使用 `Command(resume=...)` 恢复，注入人工输入并继续执行。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // 使用名为 "agent" 的已部署图表
        const assistantID = "agent";

        // 创建一个线程
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // 运行图表直到命中中断点。
        const result = await client.runs.wait(
          threadID,
          assistantID,
          { input: { "some_text": "original text" } }   # (1)!
        );

        console.log(result['__interrupt__']); // (2)!
        // > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'resumable': True,
        # >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        # >         'when': 'during'
        # >     }
        # > ]

        // 恢复图表
        console.log(await client.runs.wait(
            threadID,
            assistantID,
            // highlight-next-line
            { command: { resume: "Edited text" }}   # (3)!
        ));
        # > {'some_text': 'Edited text'}
        ```

        1. 图表以初始状态调用。
        2. 当图表命中中断点时，它会返回一个包含有效负载和元数据的中断对象。
        3. 图表使用 `{ resume: ... }` 命令对象恢复，注入人工输入并继续执行。

    === "cURL"

        创建线程：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        运行图表直到命中中断点：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"some_text\": \"original text\"}
        }"
        ```

        恢复图表：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"command\": {
            \"resume\": \"Edited text\"
          }
        }"
        ```

## 静态中断

静态中断（也称为静态断点）在节点执行之前或之后触发。

!!! warning

    不建议在人工干预工作流中使用静态中断。它们最适合用于调试和测试。

您可以通过在编译时指定 `interrupt_before` 和 `interrupt_after` 来设置静态中断：

```python
# highlight-next-line
graph = graph_builder.compile( # (1)!
    # highlight-next-line
    interrupt_before=["node_a"], # (2)!
    # highlight-next-line
    interrupt_after=["node_b", "node_c"], # (3)!
)
```

1. 断点在编译时设置。
2. `interrupt_before` 指定在节点执行前应暂停执行的节点。
3. `interrupt_after` 指定在节点执行后应暂停执行的节点。

或者，您可以在运行时设置静态中断：

=== "Python"

    ```python
    # highlight-next-line
    await client.runs.wait( # (1)!
        thread_id,
        assistant_id,
        inputs=inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
    )
    ```

    1. 调用 `client.runs.wait` 时附带 `interrupt_before` 和 `interrupt_after` 参数。这是运行时配置，可为每次调用更改。
    2. `interrupt_before` 指定在节点执行前应暂停执行的节点。
    3. `interrupt_after` 指定在节点执行后应暂停执行的节点。

=== "JavaScript"

    ```js
    // highlight-next-line
    await client.runs.wait( // (1)!
        threadID,
        assistantID,
        {
        input: input,
        // highlight-next-line
        interruptBefore: ["node_a"], // (2)!
        // highlight-next-line
        interruptAfter: ["node_b", "node_c"] // (3)!
        }
    )
    ```

    1. 调用 `client.runs.wait` 时附带 `interruptBefore` 和 `interruptAfter` 参数。这是运行时配置，可为每次调用更改。
    2. `interruptBefore` 指定在节点执行前应暂停执行的节点。
    3. `interruptAfter` 指定在节点执行后应暂停执行的节点。

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
        \"assistant_id\": \"agent\",
        \"interrupt_before\": [\"node_a\"],
        \"interrupt_after\": [\"node_b\", \"node_c\"],
        \"input\": <INPUT>
    }"
    ```

以下示例显示了如何添加静态中断：

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)

    # 使用名为 "agent" 的已部署图表
    assistant_id = "agent"

    # 创建一个线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 运行图表直到第一个断点
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input=inputs   # (1)!
    )

    # 恢复图表
    await client.runs.wait(
        thread_id,
        assistant_id,
        input=None   # (2)!
    )
    ```

    1. 图表运行直到命中第一个断点。
    2. 通过将 `input` 设置为 `None` 来恢复图表。这将运行图表直到命中下一个断点。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // 使用名为 "agent" 的已部署图表
    const assistantID = "agent";

    // 创建一个线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 运行图表直到第一个断点
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: input }   # (1)!
    );

    // 恢复图表
    await client.runs.wait(
      threadID,
      assistantID,
      { input: null }   # (2)!
    );
    ```

    1. 图表运行直到命中第一个断点。
    2. 通过将 `input` 设置为 `null` 来恢复图表。这将运行图表直到命中下一个断点。

=== "cURL"

    创建线程：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    运行图表直到断点：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <INPUT>
    }"
    ```

    恢复图表：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\"
    }"
    ```


## 了解更多

- [人工干预概念指南](../../concepts/human_in_the_loop.md): 详细了解 LangGraph 的人工干预功能。
- [常用模式](../../how-tos/human_in_the_loop/add-human-in-the-loop.md#common-patterns): 了解如何实现批准/拒绝操作、请求用户输入、工具调用审核和验证人工输入等模式。