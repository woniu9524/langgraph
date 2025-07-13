# Assistants

**Assistants** 可让您在图的核心逻辑之外单独管理配置（如 Prompt、LLM 选择、工具），从而能够快速进行更改而无需修改图的架构。这是一种通过配置变化而非结构性变化来为同一图架构创建多个专用版本的方式。

例如，设想一个构建在通用图架构上的通用写作代理。虽然结构保持不变，但不同的写作风格（如博客文章和推文）需要定制的配置来优化性能。为了支持这些变化，您可以创建多个 assistants（例如，一个用于博客，另一个用于推文），它们共享底层图，但在模型选择和系统 Prompt 上有所不同。

![assistant versions](img/assistants.png)

LangGraph Cloud API 提供了几个用于创建和管理 assistants 及其版本的端点。有关更多详细信息，请参阅[API 参考](../cloud/reference/api/api_ref.html#tag/assistants)。

!!! info

    Assistants 是 [LangGraph Platform](langgraph_platform.md) 的概念。它们在开源 LangGraph 库中不可用。

## 配置

Assistants 是在 LangGraph 开源的 [配置](low_level.md#configuration)概念的基础上构建的。
虽然开源 LangGraph 库中提供了配置，但 assistants 仅存在于 [LangGraph Platform](langgraph_platform.md) 中。这是因为 assistants 与您已部署的图紧密耦合。部署后，LangGraph Server 将使用图的默认配置设置自动为每个图创建一个默认 assistant。

在实践中，assistant 本质上是具有特定配置的图的 _实例_。因此，多个 assistants 可以引用同一个图，但可以包含不同的配置（例如，prompt、模型、工具）。LangGraph Server API 提供了几个用于创建和管理 assistants 的端点。有关如何创建 assistants 的更多详细信息，请参阅[API 参考](../cloud/reference/api/api_ref.html)和[此操作指南](../cloud/how-tos/configuration_cloud.md)。

## 版本管理

Assistants 支持版本管理以跟踪随时间发生的变化。
创建 assistant 后，对该 assistant 的后续编辑将创建新版本。有关如何管理 assistant 版本，请参阅[此操作指南](../cloud/how-tos/configuration_cloud.md#create-a-new-version-for-your-assistant)。

## 执行

**Run** 是对 assistant 的一次调用。每次 run 都可能拥有自己的输入、配置和元数据，这些都可能影响底层图的执行和输出。Run 可以选择性地在 [thread](./persistence.md#threads) 上执行。

LangGraph Platform API 提供了几个用于创建和管理 runs 的端点。有关更多详细信息，请参阅[API 参考](../cloud/reference/api/api_ref.html#tag/thread-runs/)。