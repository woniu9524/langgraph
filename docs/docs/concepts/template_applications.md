---
search:
  boost: 2
---

# 模板应用

模板是开源的参考应用，旨在帮助您快速开始使用 LangGraph 进行构建。它们提供了常见 agent 工作流的可用示例，您可以根据自己的需求进行自定义。

您可以使用 LangGraph CLI 从模板创建应用。

:::python
!!! info "需求"

    - Python >= 3.11
    - [LangGraph CLI](https://langchain-ai.github.io/langgraph/cloud/reference/cli/): 需要 langchain-cli[inmem] >= 0.1.58

## 安装 LangGraph CLI

```bash
pip install "langgraph-cli[inmem]" --upgrade
```

或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) （推荐）：

```bash
uvx --from "langgraph-cli[inmem]" langgraph dev --help
```

:::

:::js

```bash
npx @langchain/langgraph-cli --help
```

:::

## 可用模板

:::python
| 模板 | 描述 | 链接 |
| -------- | ----------- | ------ |
| **New LangGraph Project** | 一个简单的、最小化的带记忆的聊天机器人。 | [仓库](https://github.com/langchain-ai/new-langgraph-project) |
| **ReAct Agent** | 一个简单的 agent，可以灵活地扩展到许多工具。 | [仓库](https://github.com/langchain-ai/react-agent) |
| **Memory Agent** | 一个 ReAct 风格的 agent，增加了一个可在不同线程间存储记忆的工具。 | [仓库](https://github.com/langchain-ai/memory-agent) |
| **Retrieval Agent** | 一个包含基于检索的问答系统的 agent。 | [仓库](https://github.com/langchain-ai/retrieval-agent-template) |
| **Data-Enrichment Agent** | 一个执行网络搜索并将搜索结果整理成结构化格式的 agent。 | [仓库](https://github.com/langchain-ai/data-enrichment) |

:::

:::js
| 模板 | 描述 | 链接 |
| -------- | ----------- | ------ |
| **New LangGraph Project** | 一个简单的、最小化的带记忆的聊天机器人。 | [仓库](https://github.com/langchain-ai/new-langgraphjs-project) |
| **ReAct Agent** | 一个简单的 agent，可以灵活地扩展到许多工具。 | [仓库](https://github.com/langchain-ai/react-agent-js) |
| **Memory Agent** | 一个 ReAct 风格的 agent，增加了一个可在不同线程间存储记忆的工具。 | [仓库](https://github.com/langchain-ai/memory-agent-js) |
| **Retrieval Agent** | 一个包含基于检索的问答系统的 agent。 | [仓库](https://github.com/langchain-ai/retrieval-agent-template-js) |
| **Data-Enrichment Agent** | 一个执行网络搜索并将搜索结果整理成结构化格式的 agent。 | [仓库](https://github.com/langchain-ai/data-enrichment-js) |
:::

## 🌱 创建 LangGraph 应用

要从模板创建新应用，请使用 `langgraph new` 命令。

:::python

```bash
langgraph new
```

或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) （推荐）：

```bash
uvx --from "langgraph-cli[inmem]" langgraph new
```

:::

:::js

```bash
npm create langgraph
```

:::

## 后续步骤

请查看新 LangGraph 应用根目录下的 `README.md` 文件，以获取有关模板及其自定义方式的更多信息。

正确配置应用并添加 API 密钥后，您可以使用 LangGraph CLI 启动应用：

:::python

```bash
langgraph dev
```

或者通过 [`uv`](https://docs.astral.sh/uv/getting-started/installation/) （推荐）：

```bash
uvx --from "langgraph-cli[inmem]" --with-editable . langgraph dev
```

!!! info "缺少本地包？"

    如果您没有使用 `uv` 并且遇到 "`ModuleNotFoundError`" 或 "`ImportError`"，即使在安装了本地包 (`pip install -e .`) 之后，也很可能是因为您需要将 CLI 安装到本地虚拟环境中，以便 CLI "识别"本地包。您可以通过运行 `python -m pip install "langgraph-cli[inmem]"` 并重新激活您的虚拟环境，然后再运行 `langgraph dev` 来完成此操作。

:::

:::js

```bash
npx @langchain/langgraph-cli dev
```

:::

有关如何部署应用的更多信息，请参阅以下指南：

- **[启动本地 LangGraph 服务器](../tutorials/langgraph-platform/local-server.md)**：此快速入门指南展示了如何为 **ReAct Agent** 模板在本地启动 LangGraph 服务器。其他模板的步骤类似。
- **[部署到 LangGraph 平台](../cloud/quick_start.md)**：使用 LangGraph 平台部署您的 LangGraph 应用。