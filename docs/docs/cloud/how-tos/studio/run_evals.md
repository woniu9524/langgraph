# 在数据集上运行实验

LangGraph Studio 支持通过允许您在预定义 的 LangSmith 数据集上运行助手来提供评估。这使您能够了解应用程序在各种输入上的表现、将结果与参考输出进行比较，并使用 [evaluators](../../../agents/evals.md) 对结果进行评分。

本指南将向您展示如何从 Studio 端到端地运行实验。

---

## 前提条件

在运行实验之前，请确保您已具备以下条件：

1.  **LangSmith 数据集**：您的数据集应包含您要测试的输入，以及可选的用于比较的参考输出。

    -   输入的 schema 必须与助手所需的输入 schema 相匹配。有关 schema 的更多信息，请参阅 [此处](../../../concepts/low_level.md#schema)。
    -   有关创建数据集的更多信息，请参阅 [如何管理数据集](https://docs.smith.langchain.com/evaluation/how_to_guides/manage_datasets_in_application#set-up-your-dataset)。

2.  **（可选）评估器**：您可以将评估器（例如，LLM 即判官、启发式方法或自定义函数）附加到 LangSmith 中的数据集。这些评估器将在图处理完所有输入后自动运行。

    -   要了解更多信息，请阅读 [评估概念](https://docs.smith.langchain.com/evaluation/concepts#evaluators)。

3.  **正在运行的应用程序**：实验可以针对以下内容运行：
    -   部署在 [LangGraph Platform](../../quick_start.md) 上的应用程序。
    -   通过 [langgraph-cli](../../../tutorials/langgraph-platform/local-server.md) 启动的本地运行的应用程序。

---

## 分步指南

### 1. 启动实验

点击 Studio 页面右上角的 **Run experiment** 按钮。

### 2. 选择您的数据集

在出现的模态框中，选择要用于实验的数据集（或特定的数据集拆分），然后点击 **Start**。

### 3. 监控进度

现在，数据集中的所有输入都将针对活动助手运行。通过右上角的徽章监控实验的进度。

实验运行时，您可以继续在 Studio 中工作。随时点击箭头图标按钮导航到 LangSmith 并查看详细的实验结果。

---

## 故障排除

### "Run experiment" 按钮被禁用

如果 "Run experiment" 按钮被禁用，请检查以下几项：

-   **已部署的应用程序**：如果您的应用程序已部署在 LangGraph Platform 上，您可能需要创建新的版本才能启用此功能。
-   **本地开发服务器**：如果您在本地运行应用程序，请确保您已升级到最新版本的 `langgraph-cli`（`pip install -U langgraph-cli`）。此外，请确保通过在项目的 `.env` 文件中设置 `LANGSMITH_API_KEY` 来启用跟踪。

### 评估器结果缺失

当您运行实验时，任何附加的评估器都会被安排在队列中执行。如果您没有立即看到结果，那很可能是因为它们仍在等待处理。