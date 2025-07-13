---
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 评估

要评估代理的性能，您可以使用 `LangSmith` 的 [评估](https://docs.smith.langchain.com/evaluation)。您需要先定义一个评估函数来判断代理的结果，例如最终输出或轨迹。根据您的评估技术，这可能涉及参考输出，也可能不涉及：

```python
def evaluator(*, outputs: dict, reference_outputs: dict):
    # compare agent outputs against reference outputs
    output_messages = outputs["messages"]
    reference_messages = reference_outputs["messages"]
    score = compare_messages(output_messages, reference_messages)
    return {"key": "evaluator_score", "score": score}
```

要开始使用，您可以从 `AgentEvals` 包中预置评估器：

```bash
pip install -U agentevals
```

## 创建评估器

评估代理性能的一种常见方法是将其轨迹（调用工具的顺序）与参考轨迹进行比较：

```python
import json
# highlight-next-line
from agentevals.trajectory.match import create_trajectory_match_evaluator

outputs = [
    {
        "role": "assistant",
        "tool_calls": [
            {
                "function": {
                    "name": "get_weather",
                    "arguments": json.dumps({"city": "san francisco"}),
                }
            },
            {
                "function": {
                    "name": "get_directions",
                    "arguments": json.dumps({"destination": "presidio"}),
                }
            }
        ],
    }
]
reference_outputs = [
    {
        "role": "assistant",
        "tool_calls": [
            {
                "function": {
                    "name": "get_weather",
                    "arguments": json.dumps({"city": "san francisco"}),
                }
            },
        ],
    }
]

# Create the evaluator
evaluator = create_trajectory_match_evaluator(
    # highlight-next-line
    trajectory_match_mode="superset",  # (1)!
)

# Run the evaluator
result = evaluator(
    outputs=outputs, reference_outputs=reference_outputs
)
```

1. 指定轨迹将如何进行比较。如果输出轨迹是参考轨迹的超集，则 `superset` 会接受它作为有效。其他选项包括：[strict](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#strict-match)、[unordered](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#unordered-match) 和 [subset](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#subset-and-superset-match)。

下一步，了解更多关于如何[自定义轨迹匹配评估器](https://github.com/langchain-ai/agentevals?tab=readme-ov-file#agent-trajectory-match)。

### LLM 作为评判者

您可以使用 LLM 作为评判者来评估，它使用 LLM 将轨迹与参考输出进行比较并输出得分：

```python
import json
from agentevals.trajectory.llm import (
    # highlight-next-line
    create_trajectory_llm_as_judge,
    TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE
)

evaluator = create_trajectory_llm_as_judge(
    prompt=TRAJECTORY_ACCURACY_PROMPT_WITH_REFERENCE,
    model="openai:o3-mini"
)
```

## 运行评估器

要运行评估器，您首先需要创建一个 [LangSmith 数据集](https://docs.smith.langchain.com/evaluation/concepts#datasets)。要使用预置的 AgentEvals 评估器，您需要一个具有以下架构的数据集：

- **input**: `{"messages": [...]}` 用于调用代理的输入消息。
- **output**: `{"messages": [...]}` 代理输出中预期的消息历史记录。对于轨迹评估，您可以选择只保留助手消息。

```python
from langsmith import Client
from langgraph.prebuilt import create_react_agent
from agentevals.trajectory.match import create_trajectory_match_evaluator

client = Client()
agent = create_react_agent(...)
evaluator = create_trajectory_match_evaluator(...)

experiment_results = client.evaluate(
    lambda inputs: agent.invoke(inputs),
    # replace with your dataset name
    data="<Name of your dataset>",
    evaluators=[evaluator]
)
```