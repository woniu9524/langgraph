# 使用 Server API 进行时间旅行

LangGraph 提供[**时间旅行**](../../concepts/time-travel.md)功能，可以从先前的检查点恢复执行，可以选择重放相同状态或修改状态以探索其他可能性。在所有情况下，恢复过去的执行都会在历史记录中产生一个新的分支。

要使用 LangGraph Server API（通过 LangGraph SDK）进行时间旅行：

1. **运行图** 使用 [LangGraph SDK](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/) 的 [`client.runs.wait`][langgraph_sdk.client.RunsClient.wait] 或 [`client.runs.stream`][langgraph_sdk.client.RunsClient.stream] API，并提供初始输入。
2. **识别现有线程中的检查点**：使用 [`client.threads.get_history`][langgraph_sdk.client.ThreadsClient.get_history] 方法检索特定 `thread_id` 的执行历史记录，并找到所需的 `checkpoint_id`。
   或者，在希望执行暂停的节点之前设置一个[断点](./human_in_the_loop_breakpoint.md)。然后，您可以找到在该断点之前记录的最新检查点。
3. **（可选）修改图状态**：使用 [`client.threads.update_state`][langgraph_sdk.client.ThreadsClient.update_state] 方法修改检查点处的图状态，并从修改后的状态恢复执行。
4. **从检查点恢复执行**：使用 `None` 作为输入，并提供适当的 `thread_id` 和 `checkpoint_id`，调用 [`client.runs.wait`][langgraph_sdk.client.RunsClient.wait] 或 [`client.runs.stream`][langgraph_sdk.client.RunsClient.stream] API。

## 在工作流中使用时间旅行

??? example "示例图"

    ```python
    from typing_extensions import TypedDict, NotRequired
    from langgraph.graph import StateGraph, START, END
    from langchain.chat_models import init_chat_model
    from langgraph.checkpoint.memory import InMemorySaver

    class State(TypedDict):
        topic: NotRequired[str]
        joke: NotRequired[str]

    llm = init_chat_model(
        "anthropic:claude-3-7-sonnet-latest",
        temperature=0,
    )

    def generate_topic(state: State):
        """LLM 调用以生成一个笑话的主题"""
        msg = llm.invoke("Give me a funny topic for a joke")
        return {"topic": msg.content}

    def write_joke(state: State):
        """LLM 调用以根据主题写一个笑话"""
        msg = llm.invoke(f"Write a short joke about {state['topic']}")
        return {"joke": msg.content}

    # 构建工作流
    builder = StateGraph(State)

    # 添加节点
    builder.add_node("generate_topic", generate_topic)
    builder.add_node("write_joke", write_joke)

    # 添加边以连接节点
    builder.add_edge(START, "generate_topic")
    builder.add_edge("generate_topic", "write_joke")

    # 编译
    graph = builder.compile()
    ```

### 1. 运行图

=== "Python"

    ```python
    from langgraph_sdk import get_client
    client = get_client(url=<DEPLOYMENT_URL>)

    # 使用名为 "agent" 的已部署图
    assistant_id = "agent"

    # 创建一个线程
    thread = await client.threads.create()
    thread_id = thread["thread_id"]

    # 运行图
    result = await client.runs.wait(
        thread_id,
        assistant_id,
        input={}
    )
    ```

=== "JavaScript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";
    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });

    // 使用名为 "agent" 的已部署图
    const assistantID = "agent";

    // 创建一个线程
    const thread = await client.threads.create();
    const threadID = thread["thread_id"];

    // 运行图
    const result = await client.runs.wait(
      threadID,
      assistantID,
      { input: {}}
    );
    ```

=== "cURL"

    创建线程：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads \
    --header 'Content-Type: application/json' \
    --data '{}'
    ```

    运行图：

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"input\": {}
    }"
    ```

### 2. 识别检查点

=== "Python"

    ```python
    # 状态按时间倒序返回。
    states = await client.threads.get_history(thread_id)
    selected_state = states[1]
    print(selected_state)
    ```

=== "JavaScript"

    ```js
    // 状态按时间倒序返回。
    const states = await client.threads.getHistory(threadID);
    const selectedState = states[1];
    console.log(selectedState);
    ```

=== "cURL"

    ```bash
    curl --request GET \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/history \
    --header 'Content-Type: application/json'
    ```

### 3. 更新状态（可选）

`update_state` 将创建一个新的检查点。新检查点将与同一线程关联，但具有新的检查点 ID。

=== "Python"

    ```python
    new_config = await client.threads.update_state(
        thread_id,
        {"topic": "chickens"},
        # highlight-next-line
        checkpoint_id=selected_state["checkpoint_id"]
    )
    print(new_config)
    ```

=== "JavaScript"

    ```js
    const newConfig = await client.threads.updateState(
      threadID,
      {
        values: { "topic": "chickens" },
        checkpointId: selectedState["checkpoint_id"]
      }
    );
    console.log(newConfig);
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/state \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"checkpoint_id\": <CHECKPOINT_ID>,
      \"values\": {\"topic\": \"chickens\"}
    }"
    ```

### 4. 从检查点恢复执行

=== "Python"

    ```python
    await client.runs.wait(
        thread_id,
        assistant_id,
        # highlight-next-line
        input=None,
        # highlight-next-line
        checkpoint_id=new_config["checkpoint_id"]
    )
    ```

=== "JavaScript"

    ```js
    await client.runs.wait(
      threadID,
      assistantID,
      {
        // highlight-next-line
        input: null,
        // highlight-next-line
        checkpointId: newConfig["checkpoint_id"]
      }
    );
    ```

=== "cURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/threads/<THREAD_ID>/runs/wait \
    --header 'Content-Type: application/json' \
    --data "{
      \"assistant_id\": \"agent\",
      \"checkpoint_id\": <CHECKPOINT_ID>
    }"
    ```

## 了解更多

- [**LangGraph 时间旅行指南**](../../how-tos/human_in_the_loop/time-travel.md)：详细了解如何在 LangGraph 中使用时间旅行。