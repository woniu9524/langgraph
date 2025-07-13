# 如何使用 requirements.txt 设置 LangGraph 应用

LangGraph 应用必须通过 [LangGraph 配置文件](../reference/cli.md#configuration-file) 进行配置，才能部署到 LangGraph Platform（或进行自托管）。本指南将介绍如何使用 `requirements.txt` 来指定项目依赖项，从而设置一个 LangGraph 应用以进行部署。

本教程基于 [此仓库](https://github.com/langchain-ai/langgraph-example)，您可以尝试使用它来进一步了解如何设置您的 LangGraph 应用以进行部署。

!!! tip "使用 pyproject.toml 设置"
    如果您偏好使用 poetry 进行依赖管理，请查看这篇关于使用 `pyproject.toml` 设置 LangGraph Platform 的 [操作指南](./setup_pyproject.md)。

!!! tip "使用 Monorepo 设置"
    如果您有兴趣部署位于 monorepo 中的图，请查看 [此仓库](https://github.com/langchain-ai/langgraph-example-monorepo) 以了解如何操作的示例。

最终的仓库结构将如下所示：

```bash
my-app/
├── my_agent # 所有项目代码都放在这里
│   ├── utils # 图的工具集
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── requirements.txt # 包依赖
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

在每个步骤之后，都会提供一个示例文件目录，以展示如何组织代码。

## 指定依赖项

依赖项可以选择性地在以下任一文件中指定：`pyproject.toml`、`setup.py` 或 `requirements.txt`。如果未创建这些文件中的任何一个，则可以在后面的 [LangGraph 配置文件](#create-langgraph-configuration-file) 中指定依赖项。

下面的依赖项将包含在镜像中，您也可以在代码中使用它们，只要版本范围兼容即可：

```
langgraph>=0.3.27
langgraph-sdk>=0.1.66
langgraph-checkpoint>=2.0.23
langchain-core>=0.2.38
langsmith>=0.1.63
orjson>=3.9.7,<3.10.17
httpx>=0.25.0
tenacity>=8.0.0
uvicorn>=0.26.0
sse-starlette>=2.1.0,<2.2.0
uvloop>=0.18.0
httptools>=0.5.0
jsonschema-rs>=0.20.0
structlog>=24.1.0
cloudpickle>=3.0.0
```

示例 `requirements.txt` 文件：

```
langgraph
langchain_anthropic
tavily-python
langchain_community
langchain_openai

```

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都放在这里
│   └── requirements.txt # 包依赖
```

## 指定环境变量

环境变量可以选择性地在文件中指定（例如 `.env`）。请参阅 [环境变量参考](../reference/env_var.md) 来为部署配置其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
```

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都放在这里
│   └── requirements.txt # 包依赖
└── .env # 环境变量
```

## 定义图

实现您的图！图可以定义在单个文件中或多个文件中。请注意要包含在 LangGraph 应用中的每个 [CompiledStateGraph][langgraph.graph.state.CompiledStateGraph] 的变量名。在创建 [LangGraph 配置文件](../reference/cli.md#configuration-file) 时将使用这些变量名。

示例 `agent.py` 文件，展示了如何从您定义的其他模块导入（模块的代码未在此显示，请参阅 [此仓库](https://github.com/langchain-ai/langgraph-example) 查看其实现）：

```python
# my_agent/agent.py
from typing import Literal
from typing_extensions import TypedDict

from langgraph.graph import StateGraph, END, START
from my_agent.utils.nodes import call_model, should_continue, tool_node # 导入节点
from my_agent.utils.state import AgentState # 导入状态

# 定义配置
class GraphConfig(TypedDict):
    model_name: Literal["anthropic", "openai"]

workflow = StateGraph(AgentState, config_schema=GraphConfig)
workflow.add_node("agent", call_model)
workflow.add_node("action", tool_node)
workflow.add_edge(START, "agent")
workflow.add_conditional_edges(
    "agent",
    should_continue,
    {
        "continue": "action",
        "end": END,
    },
)
workflow.add_edge("action", "agent")

graph = workflow.compile()
```

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都放在这里
│   ├── utils # 图的工具集
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── requirements.txt # 包依赖
│   ├── __init__.py
│   └── agent.py # 构建图的代码
└── .env # 环境变量
```

## 创建 LangGraph 配置文件

创建一个名为 `langgraph.json` 的 [LangGraph 配置文件](../reference/cli.md#configuration-file)。有关配置文件中每个键的详细说明，请参阅 [LangGraph 配置文件参考](../reference/cli.md#configuration-file)。

示例 `langgraph.json` 文件：

```json
{
  "dependencies": ["./my_agent"],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名出现在配置文件的顶层 `graphs` 键的每个子键的值的末尾（即 `:<variable_name>`）。

!!! warning "配置文件位置"
    LangGraph 配置文件必须放置在与包含已编译图和相关依赖项的 Python 文件相同级别或更高级别的目录中。

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码都放在这里
│   ├── utils # 图的工具集
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── requirements.txt # 包依赖
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

## 下一步

在设置好项目并将其放入 GitHub 仓库后，就可以开始 [部署您的应用](./cloud.md) 了。