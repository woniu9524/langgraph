# 数据存储与隐私

本文档描述了 LangGraph CLI 和 LangGraph Server 中如何处理数据，包括内存服务器 (`langgraph dev`) 和本地 Docker 服务器 (`langgraph up`)。同时，也描述了与托管的 LangGraph Studio 前端交互时跟踪的数据。

## CLI

LangGraph **CLI** 是用于构建和运行 LangGraph 应用程序的命令行界面；请参阅 [CLI 指南](../../concepts/langgraph_cli.md) 了解更多信息。

默认情况下，大多数 CLI 命令的调用都会在调用时记录一个分析事件。这有助于我们更好地确定 CLI 体验的改进优先级。每个遥测事件包含调用进程的操作系统、操作系统版本、Python 版本、CLI 版本、命令名称（`dev`、`up`、`run` 等），以及表示是否向命令传递了标志的布尔值。您可以在此处查看完整的分析逻辑：[here](https://github.com/langchain-ai/langgraph/blob/main/libs/cli/langgraph_cli/analytics.py)。

您可以通过设置 `LANGGRAPH_CLI_NO_ANALYTICS=1` 来禁用所有 CLI 遥测。

## LangGraph Server (内存 & Docker)

[LangGraph Server](../../concepts/langgraph_server.md) 提供了一个持久化的执行运行时，它依赖于将您的应用程序状态、长期记忆、线程元数据、助手和类似资源的检查点持久化到本地文件系统或数据库。除非您已明确自定义存储位置，否则这些信息将被写入本地磁盘（对于 `langgraph dev`）或 PostgreSQL 数据库（对于 `langgraph up` 和所有部署）。

### LangSmith 跟踪

在运行 LangGraph Server（内存或 Docker）时，可以启用 LangSmith 跟踪以方便更快的调试并提供对生产环境中图状态和 LLM 提示的观察能力。您始终可以通过在服务器的运行时环境中设置 `LANGSMITH_TRACING=false` 来禁用跟踪。

### 内存开发服务器 (`langgraph dev`)

`langgraph dev` 运行一个[内存开发服务器](../../tutorials/langgraph-platform/local-server.md)，它是一个单一的 Python 进程，专为快速开发和测试而设计。它将所有检查点和内存数据保存到当前工作目录下的 `.langgraph_api` 目录中的磁盘上。除了在[CLI](#cli) 部分描述的遥测数据外，除非您启用了跟踪或图代码明确联系了外部服务，否则不会有数据离开机器。

### 独立容器 (`langgraph up`)

`langgraph up` 将您的本地包构建成 Docker 镜像，并将服务器作为[独立容器](../../concepts/deployment_options.md#standalone-container)运行，该容器由三个容器组成：API 服务器、PostgreSQL 容器和 Redis 容器。所有持久化数据（检查点、助手等）都存储在 PostgreSQL 数据库中。Redis 用于作为实时事件流的发布/订阅连接。您可以通过设置有效的 `LANGGRAPH_AES_KEY` 环境变量来加密所有保存到数据库的检查点。您还可以通过 `langgraph.json` 中为检查点和跨线程内存指定[TTL](../../how-tos/ttl/configure_ttl.md) 来控制数据的存储时间。所有持久化的线程、内存和其他数据都可以通过相关的 API 端点删除。

还会进行额外的 API 调用，以确认服务器拥有有效许可证并跟踪已执行的运行和任务数量。API 服务器会定期验证提供的许可证密钥（或 API 密钥）。

如果您禁用了[跟踪](#langsmith-tracing)，除非您的图代码明确联系了外部服务，否则不会在外部持久化用户数据。

## Studio

[LangGraph Studio](../../concepts/langgraph_studio.md) 是一个与您的 LangGraph Server 交互的图形界面。它不持久化任何私人数据（您发送到服务器的数据不会发送到 LangSmith）。虽然 Studio 界面在 [smith.langchain.com](https://smith.langchain.com) 上提供服务，但它在您的浏览器中运行，并直接连接到您的本地 LangGraph Server，因此无需将任何数据发送到 LangSmith。

如果您已登录，LangSmith 会收集一些使用情况分析数据，以帮助改善 Studio 的用户体验。这包括：

- 页面访问和导航模式
- 用户操作（按钮点击）
- 浏览器类型和版本
- 屏幕分辨率和视口大小

重要的是，不会收集任何应用程序数据或代码（或其他敏感配置详细信息）。所有这些都存储在 LangGraph Server 的持久化层中。匿名使用 Studio 时，不需要创建账户，也不会收集使用情况分析。

## 快速参考

总而言之，您可以通过关闭 CLI 分析和禁用跟踪来选择退出服务器端遥测。

| 变量                       | 目的                   | 默认                          |
| -------------------------- | ------------------------ | -------------------------------- |
| `LANGGRAPH_CLI_NO_ANALYTICS=1` | 禁用 CLI 分析     | 分析已启用                |
| `LANGSMITH_API_KEY`            | 启用 LangSmith 跟踪  | 跟踪已禁用                 |
| `LANGSMITH_TRACING=false`      | 禁用 LangSmith 跟踪 | 取决于环境           |