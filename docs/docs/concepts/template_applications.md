---
search:
  boost: 2
---

# 模板应用

模板是开源的参考应用，旨在帮助您在构建 LangGraph 应用时快速上手。它们提供了常见的代理工作流程的可用示例，可以根据您的需求进行定制。

您可以使用 LangGraph CLI 从模板创建应用。

!!! info "要求"

    - Python >= 3.11
    - [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/): 需要 langchain-cli[inmem] >= 0.1.58

## 安装 LangGraph CLI

=== "Python"

    ```bash
    pip install "langgraph-cli[inmem]" --upgrade
    ```

    或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/)（推荐）：

    ```bash
    uvx --from "langgraph-cli[inmem]" langgraph dev --help
    ```

=== "JS"

    ```bash
    npx @langchain/langgraph-cli --help
    ```

## 可用模板

| 模板                  | 描述                                                                             | Python                                                           | JS/TS                                                               |
|---------------------------|------------------------------------------------------------------------------------------|------------------------------------------------------------------|---------------------------------------------------------------------|
| **新 LangGraph 项目** | 一个简单的、最小化的带记忆的聊天机器人。                                                                          | [Repo](https://github.com/langchain-ai/new-langgraph-project)    | [Repo](https://github.com/langchain-ai/new-langgraphjs-project)     |
| **ReAct Agent**           | 一个简单的代理，可以灵活地扩展到多个工具。                                                                         | [Repo](https://github.com/langchain-ai/react-agent)              | [Repo](https://github.com/langchain-ai/react-agent-js)              |
| **Memory Agent**          | 一个 ReAct 风格的代理，增加了一个工具用于跨线程存储记忆。                                                                      | [Repo](https://github.com/langchain-ai/memory-agent)             | [Repo](https://github.com/langchain-ai/memory-agent-js)             |
| **Retrieval Agent**       | 一个包含基于检索的问答系统的代理。                                                                          | [Repo](https://github.com/langchain-ai/retrieval-agent-template) | [Repo](https://github.com/langchain-ai/retrieval-agent-template-js) |
| **Data-Enrichment Agent** | 一个执行网络搜索并将搜索结果整理成结构化格式的代理。                                                                      | [Repo](https://github.com/langchain-ai/data-enrichment)          | [Repo](https://github.com/langchain-ai/data-enrichment-js)          |


## 🌱 创建 LangGraph 应用

要从模板创建新应用，请使用 `langgraph new` 命令。

=== "Python"

    ```bash
    langgraph new
    ```

    或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/)（推荐）：

    ```bash
    uvx --from "langgraph-cli[inmem]" langgraph new
    ```

=== "JS"

    ```bash
    npm create langgraph@latest
    ```

## 后续步骤

请查看新 LangGraph 应用根目录下的 `README.md` 文件，了解有关模板及其自定义方法的更多信息。

正确配置应用并添加 API 密钥后，您可以使用 LangGraph CLI 启动应用：

=== "Python"

    ```bash
    langgraph dev
    ```

    或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/)（推荐）：

    ```bash
    uvx --from "langgraph-cli[inmem]" --with-editable . langgraph dev
    ```

    ??? info "缺少本地包？"
        如果您没有使用 `uv` 并且遇到了 "`ModuleNotFoundError`" 或 "`ImportError`"，即使已经安装了本地包 (`pip install -e .`)，您很可能需要将 CLI 安装到本地虚拟环境中，以便 CLI能够“感知”到本地包。您可以通过运行 `python -m pip install "langgraph-cli[inmem]"` 并重新激活您的虚拟环境来完成此操作，然后再运行 `langgraph dev`。

=== "JS"

    ```bash
    npx @langchain/langgraph-cli dev
    ```

有关如何部署应用的更多信息，请参阅以下指南：

- **[启动本地 LangGraph 服务器](../tutorials/langgraph-platform/local-server.md)**：本快速入门指南展示了如何为 **ReAct Agent** 模板在本地启动 LangGraph 服务器。对于其他模板，步骤类似。
- **[部署到 LangGraph 平台](../cloud/quick_start.md)**：使用 LangGraph 平台部署您的 LangGraph 应用。