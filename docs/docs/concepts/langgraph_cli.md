---
search:
  boost: 2
---

# LangGraph CLI

**LangGraph CLI** 是一个多平台命令行工具，用于在本地构建和运行 [LangGraph API server](./langgraph_server.md)。生成的服务器包含你图的所有运行、线程、助手等的 API 端点，以及运行你的代理所需的其他服务，包括用于检查点和存储的托管数据库。

## 安装

可以通过 pip 或 [Homebrew](https://brew.sh/) 安装 LangGraph CLI：

=== "pip" 
    ```bash
    pip install langgraph-cli
    ```

=== "Homebrew"
    ```bash
    brew install langgraph-cli
    ```

## 命令

LangGraph CLI 提供以下核心功能：

| 命令 | 描述 |
| -------- | -------|
| [`langgraph build`](../cloud/reference/cli.md#build) | 为可以直接部署的 [LangGraph API server](./langgraph_server.md) 构建 Docker 镜像。 |
| [`langgraph dev`](../cloud/reference/cli.md#dev) | 启动一个不需要 Docker 安装的轻量级开发服务器。此服务器非常适合快速开发和测试。此功能在版本 0.1.55 及更高版本中可用。
| [`langgraph dockerfile`](../cloud/reference/cli.md#dockerfile) | 生成一个 [Dockerfile](https://docs.docker.com/reference/dockerfile/)，可用于构建镜像和部署 [LangGraph API server](./langgraph_server.md) 实例。如果你想进一步自定义 Dockerfile 或以更自定义的方式进行部署，这将非常有用。 |
| [`langgraph up`](../cloud/reference/cli.md#up) | 在本地的 Docker 容器中启动 [LangGraph API server](./langgraph_server.md) 的一个实例。这需要本地运行 Docker 服务器。它还需要一个用于本地开发的 LangSmith API 密钥或用于生产环境的许可证密钥。 | 

有关更多信息，请参阅 [LangGraph CLI 参考](../cloud/reference/cli.md)。