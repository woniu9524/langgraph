# 迭代 Prompt

## 概述

LangGraph Studio 支持两种修改图中 Prompt 的方法：直接节点编辑和 LangSmith Playground 界面。

## 直接节点编辑

Studio 允许您直接从图界面编辑图中单个节点内部使用的 Prompt。

!!! info "先决条件"

    - [Assistants 概述](../../concepts/assistants.md)

### 图配置

定义您的[配置](https://langchain-ai.github.io/langgraph/how-tos/configuration/)，使用 `langgraph_nodes` 和 `langgraph_type` 键来指定 Prompt 字段及其关联的节点。

#### 配置参考

##### `langgraph_nodes`

- **描述**: 指定图中哪些节点与配置字段相关联。
- **值类型**: 字符串数组，其中每个字符串是图中节点的名称。
- **使用上下文**: 包含在 Pydantic 模型的 `json_schema_extra` 字典或 dataclasses 的 `metadata["json_schema_extra"]` 字典中。
- **示例**:
  ```python
  system_prompt: str = Field(
      default="You are a helpful AI assistant.",
      json_schema_extra={"langgraph_nodes": ["call_model", "other_node"]},
  )
  ```

##### `langgraph_type`

- **描述**: 指定配置字段的类型，这决定了它在 UI 中如何被处理。
- **值类型**: 字符串
- **支持的值**:
  - `"prompt"`: 表明字段包含 Prompt 文本，应在 UI 中得到特殊处理。
- **使用上下文**: 包含在 Pydantic 模型的 `json_schema_extra` 字典或 dataclasses 的 `metadata["json_schema_extra"]` 字典中。
- **示例**:
  ```python
  system_prompt: str = Field(
      default="You are a helpful AI assistant.",
      json_schema_extra={
          "langgraph_nodes": ["call_model"],
          "langgraph_type": "prompt",
      },
  )
  ```

#### 配置示例

```python
## 使用 Pydantic
from pydantic import BaseModel, Field
from typing import Annotated, Literal

class Configuration(BaseModel):
    """代理的配置。"""

    system_prompt: str = Field(
        default="You are a helpful AI assistant.",
        description="用于代理交互的系统 Prompt。 "
        "此 Prompt 设置代理的上下文和行为。",
        json_schema_extra={
            "langgraph_nodes": ["call_model"],
            "langgraph_type": "prompt",
        },
    )

    model: Annotated[
        Literal[
            "anthropic/claude-3-7-sonnet-latest",
            "anthropic/claude-3-5-haiku-latest",
            "openai/o1",
            "openai/gpt-4o-mini",
            "openai/o1-mini",
            "openai/o3-mini",
        ],
        {"__template_metadata__": {"kind": "llm"}},
    ] = Field(
        default="openai/gpt-4o-mini",
        description="用于代理主要交互的语言模型名称。 "
        "应采用以下形式：provider/model-name。",
        json_schema_extra={"langgraph_nodes": ["call_model"]},
    )

## 使用 Dataclasses
from dataclasses import dataclass, field

@dataclass(kw_only=True)
class Configuration:
    """代理的配置。"""

    system_prompt: str = field(
        default="You are a helpful AI assistant.",
        metadata={
            "description": "用于代理交互的系统 Prompt。 "
            "此 Prompt 设置代理的上下文和行为。",
            "json_schema_extra": {"langgraph_nodes": ["call_model"]},
        },
    )

    model: Annotated[str, {"__template_metadata__": {"kind": "llm"}}] = field(
        default="anthropic/claude-3-5-sonnet-20240620",
        metadata={
            "description": "用于代理主要交互的语言模型名称。 "
            "应采用以下形式：provider/model-name。",
            "json_schema_extra": {"langgraph_nodes": ["call_model"]},
        },
    )

```

### 在 UI 中编辑 Prompt

1. 找到带有关联配置字段的节点上的齿轮图标。
2. 单击以打开配置模态框。
3. 编辑值。
4. 保存以更新当前 Assistant 版本或创建新版本。

## LangSmith Playground

[LangSmith Playground](https://
docs.smith.langchain.com/prompt_engineering/how_to_guides#playground) 界面允许在不运行完整图的情况下测试单个 LLM 调用：

1. 选择一个线程。
2. 点击节点上的“View LLM Runs”。这将列出在节点内（如果有）进行的所有 LLM 调用。
3. 选择一个 LLM 运行以在 Playground 中打开。
4. 修改 Prompt 并测试不同的模型和工具设置。
5. 将更新后的 Prompt 复制回您的图。

有关更高级的 Playground 功能，请点击右上角的展开按钮。