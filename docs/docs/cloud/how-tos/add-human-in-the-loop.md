# 人工干预 (Human-in-the-loop) 使用 Server API

要审查、编辑和批准代理或工作流中的工具调用，请使用 LangGraph 的[人工干预](../../concepts/human_in_the_loop.md)功能。

## 动态中断

=== "Python"

    ```python
    from langgraph_sdk import get_client
    # highlight-next-line
    from langgraph_sdk.schema import Command
    client = get_client(url=<DEPLOYMENT_URL>)

    # 使用名为 "agent" 的已部署图
    assistant_id = "agent"

    # 创建一个线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 运行图直到命中中断。
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={"some_text": "original text"}   # (1)!
    )

    print(result['__interrupt__']) # (2)!
    # > [
    # >     {
    # >         'value': {'text_to_revise': 'original text'},
    # >         'id': '...',
    # >     }
    # > ]


    # 恢复图
    print(await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        command=Command(resume="Edited text")   # (3)!
    ))
    # > {'some_text': 'Edited text'}
    ```

    1. 使用一些初始状态调用图。
    2. 当图命中中断时，它将返回一个具有载荷和元数据的中断对象。
    3. 使用 `Command(resume=...)` 恢复图，注入用户输入并继续执行。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // 使用名为 "agent" 的已部署图
    const assistantID = "agent";

    // 创建一个线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 运行图直到命中中断。
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: { "some_text": "original text" } }   // (1)!
    );

    console.log(result['__interrupt__']); // (2)!
    // > [
    // >     {
    // >         'value': {'text_to_revise': 'original text'},
    // >         'resumable': True,
    // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
    // >         'when': 'during'
    // >     }
    // > ]

    // 恢复图
    console.log(await client.runs.wait(
        threadID,
        assistantID,
        // highlight-next-line
        { command: { resume: "Edited text" }}   // (3)!
    ));
    // > {'some_text': 'Edited text'}
    ```

    1. 使用一些初始状态调用图。
    2. 当图命中中断时，它将返回一个具有载荷和元数据的中断对象。
    3. 使用 `{ resume: ... }` 命令对象恢复图，注入用户输入并继续执行。

=== "cURL"

    创建线程：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    运行图直到命中中断：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {\"some_text\": \"original text\"}
    }"
    ```

    恢复图：

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

    这是可以在 LangGraph API 服务器中运行的示例图。
    有关更多详细信息，请参阅[LangGraph Platform 快速入门](../quick_start.md)。

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


    # 构建图
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")

    graph = graph_builder.compile()
    ```

    1. `interrupt(...)` 会在 `human_node` 处暂停执行，将指定的内容呈现给人工进行处理。
    2. 可以将任何 JSON 可序列化值传递给 `interrupt` 函数。此处传递了一个包含要修订文本的字典。
    3. 恢复后，`interrupt(...)` 的返回值是人工提供的内容，用于更新状态。

    一旦您拥有一个正在运行的 LangGraph API 服务器，您就可以使用[LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 进行交互。

    === "Python"

        ```python
        from langgraph_sdk import get_client
        # highlight-next-line
        from langgraph_sdk.schema import Command
        client = get_client(url=<DEPLOYMENT_URL>)

        # 使用名为 "agent" 的已部署图
        assistant_id = "agent"

        # 创建一个线程
        thread = await client.threads.create()
        thread_id = thread["thread_id"]

        # 运行图直到命中中断。
        result = await client.runs.wait(
            thread_id,
            assistant_id,
            input={"some_text": "original text"}   # (1)!
        )

        print(result['__interrupt__']) # (2)!
        # > [
        # >     {
        # >         'value': {'text_to_revise': 'original text'},
        # >         'id': '...',
        # >     }
        # > ]


        # 恢复图
        print(await client.runs.wait(
            thread_id,
            assistant_id,
            # highlight-next-line
            command=Command(resume="Edited text")   # (3)!
        ))
        # > {'some_text': 'Edited text'}
        ```

        1. 使用一些初始状态调用图。
        2. 当图命中中断时，它将返回一个具有载荷和元数据的中断对象。
        3. 使用 `Command(resume=...)` 恢复图，注入用户输入并继续执行。

    === "JavaScript"

        ```js
        import { Client } from "@langchain/langgraph-sdk";
        const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

        // 使用名为 "agent" 的已部署图
        const assistantID = "agent";

        // 创建一个线程
        const thread = await client.threads.create();
        const threadID = thread["thread_id"];

        // 运行图直到命中中断。
        const result = await client.runs.wait(
          threadID,
          assistantID,
          { input: { "some_text": "original text" } }   // (1)!
        );

        console.log(result['__interrupt__']); // (2)!
        // > [
        // >     {
        // >         'value': {'text_to_revise': 'original text'},
        // >         'resumable': True,
        // >         'ns': ['human_node:fc722478-2f21-0578-c572-d9fc4dd07c3b'],
        // >         'when': 'during'
        // >     }
        // > ]

        // 恢复图
        console.log(await client.runs.wait(
            threadID,
            assistantID,
            // highlight-next-line
            { command: { resume: "Edited text" }}   // (3)!
        ));
        // > {'some_text': 'Edited text'}
        ```

        1. 使用一些初始状态调用图。
        2. 当图命中中断时，它将返回一个具有载荷和元数据的中断对象。
        3. 使用 `{ resume: ... }` 命令对象恢复图，注入用户输入并继续执行。

    === "cURL"

        创建线程：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads \
        --header 'Content-Type: application/json' \
        --data '{}'
        ```

        运行图直到命中中断：

        ```bash
        curl --request POST \
        --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
        --header 'Content-Type: application/json' \
        --data "{
          \"assistant_id\": \"agent\",
          \"input\": {\"some_text\": \"original text\"}
        }"
        ```

        恢复图：

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

您可以在编译时指定 `interrupt_before` 和 `interrupt_after` 来设置静态中断：

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

    1. 调用 `client.runs.wait` 并带有 `interrupt_before` 和 `interrupt_after` 参数。这是运行时配置，每次调用都可以更改。
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

    1. 调用 `client.runs.wait` 并带有 `interruptBefore` 和 `interruptAfter` 参数。这是运行时配置，每次调用都可以更改。
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

    # 使用名为 "agent" 的已部署图
    assistant_id = "agent"

    # 创建一个线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 运行图直到第一个断点
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input=inputs   # (1)!
    )

    # 恢复图
    await client.runs.wait(
        thread_id,
        assistant_id,
        input=None   # (2)!
    )
    ```

    1. 图运行直到第一个断点。
    2. 通过为输入传递 `None` 来恢复图。这将运行图直到下一个断点。

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // 使用名为 "agent" 的已部署图
    const assistantID = "agent";

    // 创建一个线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 运行图直到断点
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: input }   // (1)!
    );

    // 恢复图
    await client.runs.wait(
      threadID,
      assistantID,
      { input: null }   // (2)!
    );
    ```

    1. 图运行直到第一个断点。
    2. 通过为输入传递 `null` 来恢复图。这将运行图直到下一个断点。

=== "cURL"

    创建线程：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    运行图直到断点：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": <INPUT>
    }"
    ```

    恢复图：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\"
    }"
    ```

## 了解更多

- [人工干预概念指南](../../concepts/human_in_the_loop.md)：了解更多关于 LangGraph 人工干预功能。
- [常见模式](../../how-tos/human_in_the_loop/add-human-in-the-loop.md#common-patterns)：了解如何实现批准/拒绝操作、请求用户输入、工具调用审查和验证用户输入等模式。