# Models

LangGraph 通过 LangChain 库内置支持 [LLMs（语言模型）](https://python.langchain.com/docs/concepts/chat_models/)。这使得将各种 LLMs 集成到你的 Agent 和工作流中变得非常容易。

## 初始化模型

使用 [`init_chat_model`](https://python.langchain.com/docs/how_to/chat_models_universal_init/) 来初始化模型：

{% include-markdown "../../snippets/chat_model_tabs.md" %}

### 直接实例化模型

如果某个模型提供商无法通过 `init_chat_model` 获取，你可以直接实例化该提供商的模型类。该模型必须实现 [BaseChatModel 接口](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html) 并支持工具调用：

```python
# Anthropic 已通过 `init_chat_model` 支持，
# 但你也可以直接实例化它。
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(
  model="claude-3-7-sonnet-latest",
  temperature=0,
  max_tokens=2048
)
```

!!! important "工具调用支持"

    如果你正在构建一个需要模型调用外部工具的 Agent 或工作流，请确保底层
    语言模型支持 [工具调用](../concepts/tools.md)。兼容模型可在 [LangChain 集成目录](https://python.langchain.com/docs/integrations/chat/) 中找到。

## 在 Agent 中使用

在使用 `create_react_agent` 时，你可以通过模型名称字符串指定模型，这是使用 `init_chat_model` 初始化模型的简写方式。这允许你在无需直接导入或实例化模型的情况下使用它。

=== "模型名称"

      ```python
      from langgraph.prebuilt import create_react_agent

      create_react_agent(
         # highlight-next-line
         model="anthropic:claude-3-7-sonnet-latest",
         # 其他参数
      )
      ```

=== "模型实例"

      ```python
      from langchain_anthropic import ChatAnthropic
      from langgraph.prebuilt import create_react_agent

      model = ChatAnthropic(
          model="claude-3-7-sonnet-latest",
          temperature=0,
          max_tokens=2048
      )
      # 或者
      # model = init_chat_model("anthropic:claude-3-7-sonnet-latest")

      agent = create_react_agent(
        # highlight-next-line
        model=model,
        # 其他参数
      )
      ```

## 高级模型配置

### 禁用流式传输

要禁用individual LLM token的流式传输，请在初始化模型时将 `disable_streaming` 设置为 `True`：

=== "`init_chat_model`"

    ```python
    from langchain.chat_models import init_chat_model

    model = init_chat_model(
        "anthropic:claude-3-7-sonnet-latest",
        # highlight-next-line
        disable_streaming=True
    )
    ```

=== "`ChatModel`"

    ```python
    from langchain_anthropic import ChatAnthropic

    model = ChatAnthropic(
        model="claude-3-7-sonnet-latest",
        # highlight-next-line
        disable_streaming=True
    )
    ```

有关 `disable_streaming` 的更多信息，请参阅[API 参考](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html#langchain_core.language_models.chat_models.BaseChatModel.disable_streaming)。

### 添加模型回退

你可以使用 `model.with_fallbacks([...])` 添加一个回退到其他模型或不同的 LLM 提供商：

=== "`init_chat_model`"

    ```python
    from langchain.chat_models import init_chat_model

    model_with_fallbacks = (
        init_chat_model("anthropic:claude-3-5-haiku-latest")
        # highlight-next-line
        .with_fallbacks([
            init_chat_model("openai:gpt-4.1-mini"),
        ])
    )
    ```

=== "`ChatModel`"

    ```python
    from langchain_anthropic import ChatAnthropic
    from langchain_openai import ChatOpenAI

    model_with_fallbacks = (
        ChatAnthropic(model="claude-3-5-haiku-latest")
        # highlight-next-line
        .with_fallbacks([
            ChatOpenAI(model="gpt-4.1-mini"),
        ])
    )
    ```

有关模型回退的更多信息，请参阅此[指南](https://python.langchain.com/docs/how_to/fallbacks/#fallback-to-better-model)。

### 使用内置速率限制器

Langchain 包含一个内置的内存速率限制器。此速率限制器是线程安全的，可以在同一进程中的多个线程之间共享。

```python
from langchain_core.rate_limiters import InMemoryRateLimiter
from langchain_anthropic import ChatAnthropic

rate_limiter = InMemoryRateLimiter(
    requests_per_second=0.1,  # <-- 非常慢！我们每 10 秒才能发起一次请求！！
    check_every_n_seconds=0.1,  # 每 100 毫秒唤醒一次，以检查是否允许发起请求，
    max_bucket_size=10,  # 控制最大突发大小。
)

model = ChatAnthropic(
   model_name="claude-3-opus-20240229",
   rate_limiter=rate_limiter
)
```

有关如何[处理速率限制](https://python.langchain.com/docs/how_to/chat_model_rate_limiting/)的信息，请参阅 LangChain 文档。

## 引入您自己的模型

如果您的目标 LLM 没有被 LangChain 正式支持，请考虑以下选项：

1. **实现自定义 LangChain 聊天模型**：创建一个符合[LangChain 聊天模型接口](https://python.langchain.com/docs/how_to/custom_chat_model/)的模型。这使得与 LangGraph 的 Agent 和工作流完全兼容，但需要对 LangChain 框架有一定的了解。

2. **自定义流式处理的直接调用**：通过 `StreamWriter` [添加自定义流式处理逻辑](../how-tos/streaming.md#use-with-any-llm)直接使用您的模型。
   有关指导，请参阅[自定义流式处理文档](../how-tos/streaming.md#use-with-any-llm)。此方法适用于不需要预构建的 Agent 集成的自定义工作流。

## 附加资源

- [多模态输入](https://python.langchain.com/docs/how_to/multimodal_inputs/)
- [结构化输出](https://python.langchain.com/docs/how_to/structured_output/)
- [模型集成目录](https://python.langchain.com/docs/integrations/chat/)
- [强制模型调用特定工具](https://python.langchain.com/docs/how_to/tool_choice/)
- [所有聊天模型操作指南](https://python.langchain.com/docs/how_to/#chat-models)
- [聊天模型集成](https://python.langchain.com/docs/integrations/chat/)