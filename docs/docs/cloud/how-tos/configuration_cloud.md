# 管理助手

在本指南中，我们将展示如何创建、配置和管理一个 [助手](../../concepts/assistants.md)。

首先，为了简要回顾配置的概念，请考虑以下简单的 `call_model` 节点和配置模式。请注意，此节点会尝试读取并使用 `config` 对象 `configurable` 定义的 `model_name`。

=== "Python"

    ```python

    class ConfigSchema(TypedDict):
        model_name: str

    builder = StateGraph(AgentState, config_schema=ConfigSchema)

    def call_model(state, config):
        messages = state["messages"]
        model_name = config.get('configurable', {}).get("model_name", "anthropic")
        model = _get_model(model_name)
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

有关配置的更多信息，[请参见此处](../../concepts/low_level.md#configuration)。

## 创建助手

### LangGraph SDK

要创建助手，请使用 [LangGraph SDK](../../concepts/sdk.md) `create` 方法。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.create) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#create) SDK 参考文档。

此示例使用与上面相同的配置模式，并创建一个助手，将 `model_name` 设置为 `openai`。

=== "Python"

    ```python
    from langgraph_sdk import get_client

    client = get_client(url=<DEPLOYMENT_URL>)
    openai_assistant = await client.assistants.create(
        # "agent"是我们部署的图的名称
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

您也可以通过 LangGraph Platform UI 创建助手。

在您的部署中，选择“助手”选项卡。这将加载您部署中所有助手（跨所有图表）的表格。

要创建新助手，请选择“+ 新建助手”按钮。这将打开一个表单，您可以在其中指定此助手所属的图表，以及提供名称、描述和基于该图表配置模式的首选配置。

要进行确认，请单击“创建助手”。这将带您进入 [LangGraph Studio](../../concepts/langgraph_studio.md)，您可以在其中测试助手。如果返回部署中的“助手”选项卡，您将在表格中看到新创建的助手。

## 使用助手

### LangGraph SDK

我们现在已经创建了一个名为“Open AI Assistant”的助手，其 `model_name` 设置为 `openai`。我们现在可以使用此助手及其配置：

=== "Python"

    ```python
    thread = await client.threads.create()
    input = {"messages": [{"role": "user", "content": "who made you?"}]}
    async for event in client.runs.stream(
        thread["thread_id"],
        # 在这里我们指定要使用的助手 ID
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
      // 在这里我们指定要使用的助手 ID
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

在您的部署中，选择“助手”选项卡。对于您希望使用的助手，单击“Studio”按钮。这将打开 LangGraph Studio 并选择该助手。当您提交输入（在图表模式或聊天模式下）时，将使用选定的助手及其配置。

## 为您的助手创建新版本

### LangGraph SDK

要编辑助手，请使用 `update` 方法。这将使用提供的编辑创建一个助手的新版本。有关更多信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.AssistantsClient.update) 和 [JS](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/js_ts_sdk_ref/#update) SDK 参考文档。

!!! note "请注意"

    您必须传递**整个**配置（如果使用，还包括元数据）。更新端点会从头开始创建新版本，而不依赖于先前版本。

例如，要更新助手系统的提示：

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

这将创建一个具有更新参数的助手新版本，并将其设置为助手的活动版本。如果您现在运行图表并传入此助手 ID，它将使用此最新版本。

### LangGraph Platform UI

您也可以通过 LangGraph Platform UI 编辑助手。

在您的部署中，选择“助手”选项卡。这将加载您部署中所有助手（跨所有图表）的表格。

要编辑现有助手，请选择指定助手的“编辑”按钮。这将打开一个表单，您可以在其中编辑助手的名称、描述和配置。

此外，如果使用 LangGraph Studio，您还可以通过“管理助手”按钮来编辑助手并创建新版本。

## 使用之前的助手版本

### LangGraph SDK

您也可以更改助手的活动版本。为此，请使用 `setLatest` 方法。

在上面的示例中，要回滚到助手的第一个版本：

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

如果您现在运行图表并传入此助手 ID，它将使用助手的第一个版本。

### LangGraph Platform UI

如果使用 LangGraph Studio，要设置助手的活动版本，请单击“管理助手”按钮并找到您希望使用的助手。选择助手和版本，然后单击“活动”切换按钮。这将更新助手，使所选版本成为活动版本。

!!! warning "删除助手"
    删除助手将删除其**所有**版本。目前无法删除单个版本，但通过将助手指向正确的版本，您可以跳过任何您不想使用的版本。