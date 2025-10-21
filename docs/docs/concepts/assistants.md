# 助手 (Assistants)

**助手 (Assistants)** 允许您将配置（例如提示、LLM 选择、工具）与图的核心逻辑分开管理，从而能够快速更改而无需修改图架构。这是一种创建同一图架构的多个专用版本的方式，每个版本通过上下文/配置的变体而不是结构性更改来针对不同的用例进行优化。

例如，设想一个构建在通用图架构上的通用写作代理。虽然结构保持不变，但不同的写作风格（例如博客文章和推文）需要量身定制的配置来优化性能。为了支持这些变体，您可以创建多个助手（例如，一个用于博客，另一个用于推文），它们共享底层图，但在模型选择和系统提示方面有所不同。

![assistant versions](img/assistants.png)

LangGraph Cloud API 提供了几个用于创建和管理助手及其版本的端点。有关更多详细信息，请参阅 [API 参考](../cloud/reference/api/api_ref.html#tag/assistants)。

!!! info

    助手是 [LangGraph Platform](langgraph_platform.md) 的一个概念。它们在开源 LangGraph 库中不可用。

## 配置

:::python
助手构建在 LangGraph 开源的配置和 [运行时上下文](low_level.md#runtime-context) 概念之上。
:::

虽然这些功能在开源 LangGraph 库中可用，但助手仅存在于 [LangGraph Platform](langgraph_platform.md) 中。这是因为助手与您部署的图紧密耦合。部署后，LangGraph Server 将使用图的默认上下文和配置设置自动为每个图创建一个默认助手。

实际上，助手只是一个具有特定配置的图的 _实例_。因此，多个助手可以引用同一个图，但包含不同的配置（例如，提示、模型、工具）。LangGraph Server API 提供了几个用于创建和管理助手的端点。有关如何创建助手的更多详细信息，请参阅 [API 参考](../cloud/reference/api/api_ref.html) 和 [此操作指南](../cloud/how-tos/configuration_cloud.md)。

## 版本控制

助手支持版本控制以跟踪随时间的变化。
创建助手后，对此助手的后续编辑将创建新版本。有关如何管理助手版本的更多详细信息，请参阅 [此操作指南](../cloud/how-tos/configuration_cloud.md#create-a-new-version-for-your-assistant)。

## 执行

一次 **运行 (run)** 是对助手的调用。每次运行可能有自己的输入、配置、上下文和元数据，这些都可能影响底层图的执行和输出。运行可以选择性地在 [线程 (thread)](./persistence.md#threads) 上执行。

LangGraph Platform API 提供了几个用于创建和管理运行的端点。有关更多详细信息，请参阅 [API 参考](../cloud/reference/api/api_ref.html#tag/thread-runs/)。