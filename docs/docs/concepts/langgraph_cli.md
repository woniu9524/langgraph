---
search:
  boost: 2
---

# LangGraph CLI

**LangGraph CLI** 是一个跨平台的命令行工具，用于在本地构建和运行 [LangGraph API 服务器](./langgraph_server.md)。生成的服务器包含了你的图（graph）运行、线程（threads）、助手（assistants）等的所有 API 端点，以及运行你的代理（agent）所需的其他服务，包括用于检查点（checkpointing）和存储的托管数据库。

:::python

## 安装

LangGraph CLI 可以通过 pip 或 [Homebrew](https://brew.sh/) 进行安装：

=== "pip"

    ```bash
    pip install langgraph-cli
    ```

=== "Homebrew"

    ```bash
    brew install langgraph-cli
    ```
:::

:::js

## 安装

LangGraph.js CLI 可以从 NPM 注册表进行安装：

=== "npx"
    ```bash
    npx @langchain/langgraph-cli
    ```

=== "npm"
    ```bash
    npm install @langchain/langgraph-cli
    ```

=== "yarn"
    ```bash
    yarn add @langchain/langgraph-cli
    ```

=== "pnpm"
    ```bash
    pnpm add @langchain/langgraph-cli
    ```

=== "bun"
    ```bash
    bun add @langchain/langgraph-cli
    ```
:::

## 命令

LangGraph CLI 提供了以下核心功能：

| 命令                                                        | 描述                                                                                                                                                                                                                                                                                                                         |
| -------------------------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| [`langgraph build`](../cloud/reference/cli.md#build)           | 构建一个可直接部署的 [LangGraph API 服务器](./langgraph_server.md) 的 Docker 镜像。                                                                                                                                                                                                                                                        |
| [`langgraph dev`](../cloud/reference/cli.md#dev)               | 启动一个无需 Docker 即可运行的轻量级开发服务器。该服务器非常适合快速开发和测试。                                                                                                                                                                                                                                                              |
| [`langgraph dockerfile`](../cloud/reference/cli.md#dockerfile) | 生成一个 [Dockerfile](https://docs.docker.com/reference/dockerfile/)，可用于构建和部署 [LangGraph API 服务器](./langgraph_server.md) 实例的镜像。如果你想进一步自定义 Dockerfile 或以更自定义的方式进行部署，这个命令会很有用。                                                                                                                                       |
| [`langgraph up`](../cloud/reference/cli.md#up)                 | 在本地的 Docker 容器中启动一个 [LangGraph API 服务器](./langgraph_server.md) 实例。这需要本地 Docker 服务正在运行。此外，还需要 LangSmith API 密钥用于本地开发，或者生产许可证密钥用于生产环境。                                                                                                                                                           |

更多信息，请参阅 [LangGraph CLI 参考文档](../cloud/reference/cli.md)。