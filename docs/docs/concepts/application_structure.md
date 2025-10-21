---
search:
  boost: 2
---

# 应用结构

## 概述

LangGraph 应用由一个或多个图、一个配置文件 (`langgraph.json`)、一个指定依赖项的文件，以及一个可选的用于指定环境变量的 `.env` 文件组成。

本指南将展示一个典型的应用文件结构，并说明如何指定使用 LangGraph Platform 部署应用所需的必要信息。

## 关键概念

要使用 LangGraph Platform 进行部署，需要提供以下信息：

1.  一个 [LangGraph 配置文件](#configuration-file-concepts) (`langgraph.json`)，用于指定应用使用的依赖项、图和环境变量。
2.  实现应用逻辑的 [图](#graphs)。
3.  一个用于指定运行应用所需的 [依赖项](#dependencies) 的文件。
4.  运行应用所需的 [环境变量](#environment-variables)。

## 文件结构

以下是应用的目录结构示例：

:::python
=== "Python (requirements.txt)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码都放在这里
    │   ├── utils # 图的工具函数
    │   │   ├── __init__.py
    │   │   ├── tools.py # 图的工具
    │   │   ├── nodes.py # 图的节点函数
    │   │   └── state.py # 图的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建图的代码
    ├── .env # 环境变量
    ├── requirements.txt # 包依赖项
    └── langgraph.json # LangGraph 的配置文件
    ```

=== "Python (pyproject.toml)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码都放在这里
    │   ├── utils # 图的工具函数
    │   │   ├── __init__.py
    │   │   ├── tools.py # 图的工具
    │   │   ├── nodes.py # 图的节点函数
    │   │   └── state.py # 图的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建图的代码
    ├── .env # 环境变量
    ├── langgraph.json  # LangGraph 的配置文件
    └── pyproject.toml # 项目的依赖项
    ```

:::

:::js

```plaintext
my-app/
├── src # 所有项目代码都放在这里
│   ├── utils # 图的可选工具函数
│   │   ├── tools.ts # 图的工具
│   │   ├── nodes.ts # 图的节点函数
│   │   └── state.ts # 图的状态定义
│   └── agent.ts # 构建图的代码
├── package.json # 包依赖项
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

:::

!!! note

    LangGraph 应用的目录结构可能因使用的编程语言和包管理器而异。

## 配置文件 {#configuration-file-concepts}

`langgraph.json` 是一个 JSON 文件，它指定了部署 LangGraph 应用所需的依赖项、图、环境变量和其他设置。

有关 JSON 文件中所有支持的键的详细信息，请参阅 [LangGraph 配置文件参考](../cloud/reference/cli.md#configuration-file)。

!!! tip

    [LangGraph CLI](./langgraph_cli.md) 默认使用当前目录下的 `langgraph.json` 配置文件。

### 示例

:::python

- 依赖项包括自定义的本地包和 `langchain_openai` 包。
- 将从文件 `./your_package/your_file.py` 中的 `variable` 变量加载单个图。
- 环境变量从 `.env` 文件加载。

```json
{
  "dependencies": ["langchain_openai", "./your_package"],
  "graphs": {
    "my_agent": "./your_package/your_file.py:agent"
  },
  "env": "./.env"
}
```

:::

:::js

- 依赖项将从本地目录中的依赖项文件（例如 `package.json`）加载。
- 将从文件 `./your_package/your_file.js` 中的 `agent` 函数加载单个图。
- `OPENAI_API_KEY` 环境变量将在内联设置。

```json
{
  "dependencies": ["."],
  "graphs": {
    "my_agent": "./your_package/your_file.js:agent"
  },
  "env": {
    "OPENAI_API_KEY": "secret-key"
  }
}
```

:::

## 依赖项

:::python
LangGraph 应用可能依赖其他 Python 包。
:::

:::js
LangGraph 应用可能依赖其他 TypeScript/JavaScript 库。
:::

通常需要提供以下信息才能正确设置依赖项：

:::python

1.  目录中用于指定依赖项的文件（例如 `requirements.txt`、`pyproject.toml` 或 `package.json`）。
   :::

:::js

1.  目录中用于指定依赖项的文件（例如 `package.json`）。
   :::

2.  [LangGraph 配置文件](#configuration-file-concepts)中的 `dependencies` 键，用于指定运行 LangGraph 应用所需的依赖项。
3.  任何额外的二进制文件或系统库都可以使用 [LangGraph 配置文件](#configuration-file-concepts)中的 `dockerfile_lines` 键进行指定。

## 图

使用 [LangGraph 配置文件](#configuration-file-concepts)中的 `graphs` 键来指定将在部署的 LangGraph 应用中提供的图。

可以在配置文件中指定一个或多个图。每个图由一个唯一的名称标识，以及一个用于指定（1）已编译图或（2）定义图的函数的路径。

## 环境变量

如果要在本地使用已部署的 LangGraph 应用，可以在 [LangGraph 配置文件](#configuration-file-concepts) 的 `env` 键中配置环境变量。

对于生产部署，通常需要在部署环境中配置环境变量。