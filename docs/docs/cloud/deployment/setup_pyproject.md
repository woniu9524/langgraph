# 如何使用 `pyproject.toml` 设置 LangGraph 应用

LangGraph 应用必须使用 [LangGraph 配置文件](../reference/cli.md#configuration-file) 进行配置，才能部署到 LangGraph 平台（或自托管）。本操作指南将介绍如何使用 `pyproject.toml` 来定义包的依赖项，从而设置用于部署的 LangGraph 应用的基本步骤。

本次演练基于 [此仓库](https://github.com/langchain-ai/langgraph-example-pyproject)，您可以尝试使用它来进一步了解如何设置 LangGraph 应用以进行部署。

!!! tip "使用 `requirements.txt` 设置"
    如果您更倾向于使用 `requirements.txt` 来管理依赖项，请查看 [本操作指南](./setup.md)。

!!! tip "使用 Monorepo 设置"
    如果您有兴趣部署位于 monorepo 中的图，请查看 [此仓库](https://github.com/langchain-ai/langgraph-example-monorepo) 以了解如何操作。

最终的仓库结构将如下所示：

```bash
my-app/
├── my_agent # 所有项目代码均在此处
│   ├── utils # 图的工具集
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
├── langgraph.json  # LangGraph 的配置文件
└── pyproject.toml # 项目的依赖项
```

在每个步骤之后，都会提供一个示例文件目录，以演示代码的组织方式。

## 指定依赖项

依赖项可以根据需要指定在以下任何文件中：`pyproject.toml`、`setup.py` 或 `requirements.txt`。如果未创建这些文件中的任何一个，则稍后可以在 [LangGraph 配置文件](#create-langgraph-configuration-file) 中指定依赖项。

以下依赖项将包含在镜像中，您也可以在代码中使用它们，只要版本范围兼容即可：

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

示例 `pyproject.toml` 文件：

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[project]
name = "my-agent"
version = "0.0.1"
description = "An excellent agent build for LangGraph Platform."
authors = [
    {name = "Polly the parrot", email = "1223+polly@users.noreply.github.com"}
]
license = {text = "MIT"}
readme = "README.md"
requires-python = ">=3.9"
dependencies = [
    "langgraph>=0.2.0",
    "langchain-fireworks>=0.1.3"
]

[tool.hatch.build.targets.wheel]
packages = ["my_agent"]
```

示例文件目录：

```bash
my-app/
└── pyproject.toml   # 您的图所需的 Python 包
```

## 指定环境变量

环境变量可以根据需要指定在一个文件中（例如 `.env`）。请参阅 [环境变量参考](../reference/env_var.md) 以配置部署的其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
FIREWORKS_API_KEY=key
```

示例文件目录：

```bash
my-app/
├── .env # 包含环境变量的文件
└── pyproject.toml
```

## 定义图

实现您的图！图可以定义在单个文件中或多个文件中。请注意要包含在 LangGraph 应用中的每个 [CompiledStateGraph][langgraph.graph.state.CompiledStateGraph] 的变量名。这些变量名将在稍后用于创建 [LangGraph 配置文件](../reference/cli.md#configuration-file)。

示例 `agent.py` 文件，展示了如何从您定义的其他模块导入（模块代码未显示在此处，请参阅 [此仓库](https://github.com/langchain-ai/langgraph-example-pyproject) 查看其实现）：

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
├── my_agent # 所有项目代码均在此处
│   ├── utils # 图的工具集
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env
└── pyproject.toml
```

## 创建 LangGraph 配置文件

创建一个名为 `langgraph.json` 的 [LangGraph 配置文件](../reference/cli.md#configuration-file)。有关配置文件中每个键的详细解释，请参阅 [LangGraph 配置文件参考](../reference/cli.md#configuration-file)。

示例 `langgraph.json` 文件：

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./my_agent/agent.py:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名出现在配置文件的顶级 `graphs` 键的每个子键值之后（例如 `:<variable_name>`）。

!!! warning "配置文件位置"
    LangGraph 配置文件必须放置在与包含已编译图和相关依赖项的 Python 文件相同级别或更高级别的目录中。

示例文件目录：

```bash
my-app/
├── my_agent # 所有项目代码均在此处
│   ├── utils # 图的工具集
│   │   ├── __init__.py
│   │   ├── tools.py # 图的工具
│   │   ├── nodes.py # 图的节点函数
│   │   └── state.py # 图的状态定义
│   ├── __init__.py
│   └── agent.py # 构建图的代码
├── .env # 环境变量
├── langgraph.json  # LangGraph 的配置文件
└── pyproject.toml # 项目的依赖项
```

## 下一步

设置好项目并将其放置在 GitHub 仓库中后，就可以 [部署您的应用](./cloud.md) 了。