# 模型

LangGraph 通过 LangChain 库内置支持 [LLM（语言模型）](https://python.langchain.com/docs/concepts/chat_models/)。这使得将各种 LLM 集成到您的代理和工作流程中变得更加容易。

## 初始化模型

:::python
使用 [`init_chat_model`](https://python.langchain.com/docs/how_to/chat_models_universal_init/) 来初始化模型：

{% include-markdown "../../snippets/chat_model_tabs.md" %}
:::

:::js
使用模型提供程序类来初始化模型：

=== "OpenAI"

    ```typescript
    import { ChatOpenAI } from "@langchain/openai";

    const model = new ChatOpenAI({
      model: "gpt-4o",
      temperature: 0,
    });
    ```

=== "Anthropic"

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";

    const model = new ChatAnthropic({
      model: "claude-3-5-sonnet-20240620",
      temperature: 0,
      maxTokens: 2048,
    });
    ```

=== "Google"

    ```typescript
    import { ChatGoogleGenerativeAI } from "@langchain/google-genai";

    const model = new ChatGoogleGenerativeAI({
      model: "gemini-1.5-pro",
      temperature: 0,
    });
    ```

=== "Groq"

    ```typescript
    import { ChatGroq } from "@langchain/groq";

    const model = new ChatGroq({
      model: "llama-3.1-70b-versatile",
      temperature: 0,
    });
    ```

:::

:::python

### 直接实例化模型

如果某个模型提供程序无法通过 `init_chat_model` 获取，您可以直接实例化该提供程序的模型类。模型必须实现 [BaseChatModel 接口](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html)并支持工具调用：

```python
# Anthropic 已通过 `init_chat_model` 支持，
# 但您也可以直接实例化它。
from langchain_anthropic import ChatAnthropic

model = ChatAnthropic(
  model="claude-3-7-sonnet-latest",
  temperature=0,
  max_tokens=2048
)
```

:::

!!! important "Tool calling support"

    如果您正在构建需要模型调用外部工具的代理或工作流，请确保底层
    语言模型支持 [工具调用](../concepts/tools.md)。兼容模型可在 [LangChain 集成目录](https://python.langchain.com/docs/integrations/chat/) 中找到。

## 在代理中使用

:::python
当使用 `create_react_agent` 时，您可以通过其名称字符串指定模型，这是使用 `init_chat_model` 初始化模型的简写。这允许您在无需导入或直接实例化模型的情况下使用它。

=== "model name"

      ```python
      from langgraph.prebuilt import create_react_agent

      create_react_agent(
         # highlight-next-line
         model="anthropic:claude-3-7-sonnet-latest",
         # other parameters
      )
      ```

=== "model instance"

      ```python
      from langchain_anthropic import ChatAnthropic
      from langgraph.prebuilt import create_react_agent

      model = ChatAnthropic(
          model="claude-3-7-sonnet-latest",
          temperature=0,
          max_tokens=2048
      )
      # Alternatively
      # model = init_chat_model("anthropic:claude-3-7-sonnet-latest")

      agent = create_react_agent(
        # highlight-next-line
        model=model,
        # other parameters
      )
      ```

:::

:::js
当使用 `createReactAgent` 时，您可以直接传递模型实例：

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const model = new ChatOpenAI({
  model: "gpt-4o",
  temperature: 0,
});

const agent = createReactAgent({
  llm: model,
  tools: tools,
});
```

:::

:::python

### 动态模型选择

将可调用函数传递给 `create_react_agent` 以在运行时动态选择模型。这对于您希望根据用户输入、配置设置或其他运行时条件选择模型的场景非常有用。

选择器函数必须返回一个聊天模型。如果您正在使用工具，则必须在选择器函数中将工具绑定到该模型。

  ```python
from dataclasses import dataclass
from typing import Literal
from langchain.chat_models import init_chat_model
from langchain_core.language_models import BaseChatModel
from langchain_core.tools import tool
from langgraph.prebuilt import create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState
from langgraph.runtime import Runtime

@tool
def weather() -> str:
    """Returns the current weather conditions."""
    return "It's nice and sunny."


# Define the runtime context
@dataclass
class CustomContext:
    provider: Literal["anthropic", "openai"]

# Initialize models
openai_model = init_chat_model("openai:gpt-4o")
anthropic_model = init_chat_model("anthropic:claude-sonnet-4-20250514")


# Selector function for model choice
def select_model(state: AgentState, runtime: Runtime[CustomContext]) -> BaseChatModel:
    if runtime.context.provider == "anthropic":
        model = anthropic_model
    elif runtime.context.provider == "openai":
        model = openai_model
    else:
        raise ValueError(f"Unsupported provider: {runtime.context.provider}")

    # With dynamic model selection, you must bind tools explicitly
    return model.bind_tools([weather])


# Create agent with dynamic model selection
agent = create_react_agent(select_model, tools=[weather])

# Invoke with context to select model
output = agent.invoke(
    {
        "messages": [
            {
                "role": "user",
                "content": "Which model is handling this?",
            }
        ]
    },
    context=CustomContext(provider="openai"),
)

print(output["messages"][-1].text())
```

!!! version-added "Added in version 0.6.0"

:::

## 高级模型配置

### 禁用流式传输

:::python
要禁用单个 LLM token 的流式传输，请在初始化模型时设置 `disable_streaming=True`：

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

有关 `disable_streaming` 的更多信息，请参阅 [API 参考](https://python.langchain.com/api_reference/core/language_models/langchain_core.language_models.chat_models.BaseChatModel.html#langchain_core.language_models.chat_models.BaseChatModel.disable_streaming)。
:::

:::js
要禁用单个 LLM token 的流式传输，请在初始化模型时设置 `streaming: false`：

```typescript
import { ChatOpenAI } from "@langchain/openai";

const model = new ChatOpenAI({
  model: "gpt-4o",
  streaming: false,
});
```

:::

### 添加模型回退

:::python
您可以使用 `model.with_fallbacks([...])` 添加对不同模型或不同 LLM 提供程序的后备支持：

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

有关模型回退的更多信息，请参阅此 [指南](https://python.langchain.com/docs/how_to/fallbacks/#fallback-to-better-model)。
:::

:::js
您可以使用 `model.withFallbacks([...])` 添加对不同模型或不同 LLM 提供程序的后备支持：

```typescript
import { ChatOpenAI } from "@langchain/openai";
import { ChatAnthropic } from "@langchain/anthropic";

const modelWithFallbacks = new ChatOpenAI({
  model: "gpt-4o",
}).withFallbacks([
  new ChatAnthropic({
    model: "claude-3-5-sonnet-20240620",
  }),
]);
```

有关模型后备的更多信息，请参阅此 [指南](https://js.langchain.com/docs/how_to/fallbacks/#fallback-to-better-model)。
:::

:::python

### 使用内置速率限制器

Langchain 包含一个内置的内存速率限制器。此速率限制器是线程安全的，并且可以被同一进程中的多个线程共享。

```python
from langchain_core.rate_limiters import InMemoryRateLimiter
from langchain_anthropic import ChatAnthropic

rate_limiter = InMemoryRateLimiter(
    requests_per_second=0.1,  # <-- Super slow! We can only make a request once every 10 seconds!!
    check_every_n_seconds=0.1,  # Wake up every 100 ms to check whether allowed to make a request,
    max_bucket_size=10,  # Controls the maximum burst size.
)

model = ChatAnthropic(
   model_name="claude-3-opus-20240229",
   rate_limiter=rate_limiter
)
```

有关如何[处理速率限制](https://python.langchain.com/docs/how_to/chat_model_rate_limiting/) 的更多信息，请参阅 LangChain 文档。
:::

## 引入您自己的模型

如果 LangChain 不支持您想要的 LLM，请考虑以下选项：

:::python

1. **实现自定义 LangChain 聊天模型**：创建一个符合 [LangChain 聊天模型接口](https://python.langchain.com/docs/how_to/custom_chat_model/) 的模型。这使得可以与 LangGraph 的代理和工作流完全兼容，但需要了解 LangChain 框架。

   :::

:::js

1. **实现自定义 LangChain 聊天模型**：创建一个符合 [LangChain 聊天模型接口](https://js.langchain.com/docs/how_to/custom_chat/) 的模型。这使得可以与 LangGraph 的代理和工作流完全兼容，但需要了解 LangChain 框架。

   :::

2. **使用自定义流逻辑直接调用**：通过 [添加自定义流逻辑](../how-tos/streaming.md#use-with-any-llm) 和 `StreamWriter` 直接调用您的模型。
   有关指导，请参阅 [自定义流文档](../how-tos/streaming.md#use-with-any-llm)。此方法适用于不需要预构建代理集成的自定义工作流。

## 附加资源

:::python

- [Multimodal inputs](https://python.langchain.com/docs/how_to/multimodal_inputs/)
- [Structured outputs](https://python.langchain.com/docs/how_to/structured_output/)
- [Model integration directory](https://python.langchain.com/docs/integrations/chat/)
- [Force model to call a specific tool](https://python.langchain.com/docs/how_to/tool_choice/)
- [All chat model how-to guides](https://python.langchain.com/docs/how_to/#chat-models)
- [Chat model integrations](https://python.langchain.com/docs/integrations/chat/)

  :::

:::js

- [Multimodal inputs](https://js.langchain.com/docs/how_to/multimodal_inputs/)
- [Structured outputs](https://js.langchain.com/docs/how_to/structured_output/)
- [Model integration directory](https://js.langchain.com/docs/integrations/chat/)
- [Force model to call a specific tool](https://js.langchain.com/docs/how_to/tool_choice/)
- [All chat model how-to guides](https://js.langchain.com/docs/how_to/#chat-models)
- [Chat model integrations](https://js.langchain.com/docs/integrations/chat/)

  :::