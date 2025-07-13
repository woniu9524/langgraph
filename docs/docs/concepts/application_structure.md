---
search:
  boost: 2
---

# 应用结构

## 概述

LangGraph 应用由一个或多个 graph、一个配置文件 (`langgraph.json`)、一个指定依赖项的文件以及一个可选的 `.env` 文件（用于指定环境变量）组成。

本指南将展示一个典型的应用结构，并说明如何指定使用 LangGraph Platform 部署应用所需的必要信息。

## 核心概念

要使用 LangGraph Platform 进行部署，应提供以下信息：

1.  一个 LangGraph 配置文件 (`langgraph.json`)，用于指定应用要使用的依赖项、图和环境变量。
2.  实现应用逻辑的图。
3.  一个指定运行应用所需的依赖项的文件。
4.  运行应用所需的环境变量。

## 文件结构

以下是 Python 和 JavaScript 应用的目录结构示例：

=== "Python (requirements.txt)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码均在此目录下
    │   ├── utils # graph 的辅助工具
    │   │   ├── __init__.py
    │   │   ├── tools.py # graph 的工具
    │   │   ├── nodes.py # graph 的节点函数
    │   │   └── state.py # graph 的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建 graph 的代码
    ├── .env # 环境变量
    ├── requirements.txt # 包依赖项
    └── langgraph.json # LangGraph 的配置文件
    ```
=== "Python (pyproject.toml)"

    ```plaintext
    my-app/
    ├── my_agent # 所有项目代码均在此目录下
    │   ├── utils # graph 的辅助工具
    │   │   ├── __init__.py
    │   │   ├── tools.py # graph 的工具
    │   │   ├── nodes.py # graph 的节点函数
    │   │   └── state.py # graph 的状态定义
    │   ├── __init__.py
    │   └── agent.py # 构建 graph 的代码
    ├── .env # 环境变量
    ├── langgraph.json  # LangGraph 的配置文件
    └── pyproject.toml # 项目的依赖项
    ```

=== "JS (package.json)"

    ```plaintext
    my-app/
    ├── src # 所有项目代码均在此目录下
    │   ├── utils # 可选的 graph 辅助工具
    │   │   ├── tools.ts # graph 的工具
    │   │   ├── nodes.ts # graph 的节点函数
    │   │   └── state.ts # graph 的状态定义
    │   └── agent.ts # 构建 graph 的代码
    ├── package.json # 包依赖项
    ├── .env # 环境变量
    └── langgraph.json # LangGraph 的配置文件
    ```

!!! note

    LangGraph 应用的目录结构可能会因所使用的编程语言和包管理器而异。

## 配置文件 {#configuration-file-concepts}

`langgraph.json` 是一个 JSON 文件，用于指定部署 LangGraph 应用所需的依赖项、图、环境变量和其他设置。

有关 JSON 文件中所有支持的键的详细信息，请参阅 [LangGraph 配置文件参考](../cloud/reference/cli.md#configuration-file)。

!!! tip

    [LangGraph CLI](./langgraph_cli.md) 默认使用当前目录中的 `langgraph.json` 配置文件。

### 示例

=== "Python"

    *   依赖项包括一个自定义本地包和 `langchain_openai` 包。
    *   将从文件 `./your_package/your_file.py` 中的 `variable` 加载一个图。
    *   环境变量将从 `.env` 文件加载。

    ```json
    {
        "dependencies": [
            "langchain_openai",
            "./your_package"
        ],
        "graphs": {
            "my_agent": "./your_package/your_file.py:agent"
        },
        "env": "./.env"
    }
    ```

=== "JavaScript"

    *   依赖项将从本地目录中的依赖项文件（例如 `package.json`）加载。
    *   将从文件 `./your_package/your_file.js` 中的 `agent` 函数加载一个图。
    *   环境变量 `OPENAI_API_KEY` 内联设置。

    ```json
    {
        "dependencies": [
            "."
        ],
        "graphs": {
            "my_agent": "./your_package/your_file.js:agent"
        },
        "env": {
            "OPENAI_API_KEY": "secret-key"
        }
    }
    ```

## 依赖项

LangGraph 应用可能依赖于其他 Python 包或 JavaScript 库（取决于应用是用哪种编程语言编写的）。

通常需要提供以下信息才能正确设置依赖项：

1.  目录中指定依赖项的文件（例如 `requirements.txt`、`pyproject.toml` 或 `package.json`）。
2.  [LangGraph 配置文件](#configuration-file-concepts)中的 `dependencies` 键，用于指定运行 LangGraph 应用所需的依赖项。
3.  可以使用 [LangGraph 配置文件](#configuration-file-concepts)中的 `dockerfile_lines` 键指定任何其他二进制文件或系统库。

## 图

使用[LangGraph 配置文件](#configuration-file-concepts)中的 `graphs` 键来指定将在部署的 LangGraph 应用中提供的图。

您可以在配置文件中指定一个或多个图。每个图都由一个名称（应是唯一的）和一个路径标识，该路径指向：(1) 编译后的图或 (2) 定义图的函数。

## 环境变量

如果您在本地使用已部署的 LangGraph 应用，可以在[LangGraph 配置文件](#configuration-file-concepts)的 `env` 键中配置环境变量。

对于生产部署，您通常希望在部署环境中配置环境变量。