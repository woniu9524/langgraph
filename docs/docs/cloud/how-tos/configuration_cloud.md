# 管理助手

在本指南中，我们将演示如何创建、配置和管理一个[助手](../../concepts/assistants.md)。

首先，简单回顾一下运行时上下文（runtime context）的概念，来看下面这个简单的 `call_model` 节点和上下文模式（context schema）。请注意，这个节点试图读取并使用 `Runtime` 对象 `context` 属性定义的 `model_provider`。

=== "Python"

    ```python
    @dataclass
    class ContextSchema:
        llm_provider: str = "anthropic"

    builder = StateGraph(AgentState, context_schema=ContextSchema)

    def call_model(state, runtime: Runtime[ContextSchema]):
        messages = state["messages"]
        model = _get_model(runtime.context.llm_provider)
        response = model.invoke(messages)
        # 我们返回一个列表，因为它将被添加到现有列表中
        return {"messages": [response]}
    ```

=== "Javascript"

    ```js
    import { Annotation } from "@langchain/langgraph";

    const ConfigSchema = Annotation.Root({
        model_name: Annotation<string>,
        system_prompt:
    });

    const builder = new StateGraph(AgentState, ConfigSchema)

    function callModel(state: State, config: RunnableConfig) {
      const messages = state.messages;
      const modelName = config.configurable?.model_name ?? "anthropic";
      const model = _getModel(modelName);
      const response = model.invoke(messages);
      // 我们返回一个列表，因为它将被添加到现有列表中
      return { messages: [response] };
    }
    ```

:::python
有关运行时上下文的更多信息，[请参见此处](../../concepts/low_level.md#runtime-context)。
:::

## 创建助手

### LangGraph SDK

要创建助手，请使用 [LangGraph SDK](../../concepts/sdk.md) 的 `create` 方法。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.create) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create) SDK 参考文档。

本示例使用与上面相同的配置模式，并创建一个 `model_name` 设置为 `openai` 的助手。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    openai_assistant = await client.assistants.create(
        # "agent" 是我们部署的一个图的名称
        "agent", config={"configurable": {"model_name": "openai"}}, name="Open AI Assistant"
    )

    print(openai_assistant)
    ```

=== "Javascript"

    ```js
    import { Client } from "@langchain/langgraph-sdk";

    const client = new Client({ apiUrl: <DEPLOYMENT_URL> });
    const openAIAssistant = await client.assistants.create({
        graphId: 'agent',
        name: "Open AI Assistant",
        config: { "configurable": { "model_name": "openai" } },
    });

    console.log(openAIAssistant);
    ```

=== "CURL"

    ```bash
    curl --request POST \
        --url <DEPLOYMENT_URL>/assistants \
        --header 'Content-Type: application/json' \
        --data '{"graph_id":"agent", "config":{"configurable":{"model_name":"openai"}}, "name": "Open AI Assistant"}'
    ```

输出：

    {
        "assistant_id": "62e209ca-9154-432a-b9e9-2d75c7a9219b",
        "graph_id": "agent",
        "name": "Open AI Assistant"
        "config": {
            "configurable": {
                "model_name": "openai"
            }
        },
        "metadata": {}
        "created_at": "2024-08-31T03:09:10.230718+00:00",
        "updated_at": "2024-08-31T03:09:10.230718+00:00",
    }

### LangGraph Platform UI

您也可以从 LangGraph Platform UI 创建助手。

在您的部署中，选择“Assistants”（助手）选项卡。这将加载一个包含您部署中所有助手（跨所有图）的表格。

要创建新助手，请选择“+ New assistant”（+ 新建助手）按钮。这将打开一个表单，您可以在其中指定该助手使用的图，以及提供名称、描述和基于该图配置模式所需的配置。

要确认，请点击“Create assistant”（创建助手）。这将带您到 [LangGraph Studio](../../concepts/langgraph_studio.md)，您可以在那里测试助手。如果您回到部署中的“Assistants”（助手）选项卡，您将看到新创建的助手出现在表格中。

## 使用助手

### LangGraph SDK

我们现在已经创建了一个名为“Open AI Assistant”的助手，其 `model_name` 设置为 `openai`。我们现在可以使用此助手及其配置：

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        # 这里是我们指定要使用的助手 ID
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
    const input = { "messages": [{ "role": "user", "content": "who made you?" }] };

    const streamResponse = client.runs.stream(
      thread["thread_id"],
      // 这里是我们指定要使用的助手 ID
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

    ```
    Receiving event of type: metadata
    {'run_id': '1ef6746e-5893-67b1-978a-0f1cd4060e16'}



    Receiving event of type: updates
    {'agent': {'messages': [{'content': 'I was created by OpenAI, a research organization focused on developing and advancing artificial intelligence technology.', 'additional_kwargs': {}, 'response_metadata': {'finish_reason': 'stop', 'model_name': 'gpt-4o-2024-05-13', 'system_fingerprint': 'fp_157b3831f5'}, 'type': 'ai', 'name': None, 'id': 'run-e1a6b25c-8416-41f2-9981-f9cfe043f414', 'example': False, 'tool_calls': [], 'invalid_tool_calls': [], 'usage_metadata': None}]}}
    ```

### LangGraph Platform UI

在您的部署中，选择“Assistants”（助手）选项卡。对于您想要使用的助手，点击“Studio”按钮。这将打开 LangGraph Studio 并显示选定的助手。当您提交输入（无论是使用 Graph 模式还是 Chat 模式）时，将使用选定的助手及其配置。

## 为您的助手创建新版本

### LangGraph SDK

要编辑助手，请使用 `update` 方法。这将创建一个包含所提供编辑内容的新助手版本。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.update) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#update) SDK 参考文档。

!!! note "注意"

    您必须传入**完整的**配置（如果使用，还包括元数据）。更新端点会从头开始创建新版本，而不依赖于先前版本。

例如，要更新助手的系统提示：

=== "Python"

    ```python
    openai_assistant_v2 = await client.assistants.update(
        openai_assistant["assistant_id"],
        config={
            "configurable": {
                "model_name": "openai",
                "system_prompt": "You are an unhelpful assistant!",
            }
        },
    )
    ```

=== "Javascript"

    ```js
    const openaiAssistantV2 = await client.assistants.update(
        openai_assistant["assistant_id"],
        {
            config: {
                configurable: {
                    model_name: 'openai',
                    system_prompt: 'You are an unhelpful assistant!',
                },
        },
    });
    ```

=== "CURL"

    ```bash
    curl --request PATCH \
    --url <DEPOLYMENT_URL>/assistants/<ASSISTANT_ID> \
    --header 'Content-Type: application/json' \
    --data '{
    "config": {"model_name": "openai", "system_prompt": "You are an unhelpful assistant!"}
    }'
    ```

这将使用更新后的参数创建一个助手的新版本，并将其设置为助手的活动版本。如果您现在运行图并将此助手 ID 传入，它将使用此最新版本。

### LangGraph Platform UI

您也可以从 LangGraph Platform UI 编辑助手。

在您的部署中，选择“Assistants”（助手）选项卡。这将加载一个包含您部署中所有助手（跨所有图）的表格。

要编辑现有助手，请为您指定的助手选择“Edit”（编辑）按钮。这将打开一个表单，您可以在其中编辑助手的名称、描述和配置。

此外，如果您使用 LangGraph Studio，可以通过“Manage Assistants”（管理助手）按钮来编辑助手并创建新版本。

## 使用之前的助手版本

### LangGraph SDK

您还可以更改助手的活动版本。为此，请使用 `set_latest` 方法。

在上例中，要回滚到助手的第一个版本：

=== "Python"

    ```python
    await client.assistants.set_latest(openai_assistant['assistant_id'], 1)
    ```

=== "Javascript"

    ```js
    await client.assistants.setLatest(openaiAssistant['assistant_id'], 1);
    ```

=== "CURL"

    ```bash
    curl --request POST \
    --url <DEPLOYMENT_URL>/assistants/<ASSISTANT_ID>/latest \
    --header 'Content-Type: application/json' \
    --data '{
    "version": 1
    }'
    ```

如果您现在运行图并将此助手 ID 传入，它将使用助手的第一个版本。

### LangGraph Platform UI

如果您使用 LangGraph Studio，要设置助手的活动版本，请点击“Manage Assistants”（管理助手）按钮并找到您想要使用的助手。选择助手和版本，然后点击“Active”（活动）切换按钮。这将更新助手，使选定的版本成为活动的。

!!! warning "删除助手"
删除助手将删除其**所有**版本。目前没有办法删除单个版本，但通过将您的助手指向正确的版本，您可以跳过任何您不希望使用的版本。