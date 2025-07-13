# 无效许可证

当尝试启动自托管的 LangGraph Platform 服务器时，许可证验证失败会引发此错误。此错误特定于 LangGraph Platform，与开源库无关。

## 发生情况

在运行自托管的 LangGraph Platform 部署而没有有效企业许可证或 API 密钥时，会出现此错误。

## 故障排除

### 确认部署类型

首先，确认所需的部署模式。

#### 用于本地开发

如果您仅在本地进行开发，可以通过运行 `langgraph dev` 来使用轻量级的内存服务器。
有关更多信息，请参阅[本地服务器](../../tutorials/langgraph-platform/local-server.md)文档。

#### 用于托管的 LangGraph Platform

如果您需要一个快速的托管环境，请考虑使用[云 SaaS](../../concepts/langgraph_cloud.md) 部署选项。这不需要额外的许可证密钥。

#### 用于独立容器（Lite）

如果您的部署每年节点执行次数可能不会超过 100 万次，并且不需要 Crons 和其他企业功能，请考虑[独立容器](../../concepts/deployment_options.md)部署选项。

您可以通过在环境中（例如，在 `langgraph.json` 引用的 `.env` 文件中）设置有效的 `LANGSMITH_API_KEY` 并构建 Docker 镜像来部署独立容器。API 密钥必须与**Plus**或更高计划的账户相关联。

#### 用于独立容器（Enterprise）

要进行完全自托管，请设置 `LANGGRAPH_CLOUD_LICENSE_KEY` 环境变量。如果您对企业许可证密钥感兴趣，请联系 LangChain 支持团队。

有关部署选项及其功能的更多信息，请参阅[部署选项](../../concepts/deployment_options.md)文档。

### 确认凭据

如果您已确认要自托管 LangGraph Platform，请验证您的凭据。

#### 用于独立容器（Lite）

1. 确认您已在部署环境或 `.env` 文件中提供了有效的 `LANGSMITH_API_KEY` 环境变量。
2. 确认提供的 API 密钥与**Plus**或**Enterprise**计划（或同等计划）的账户相关联。

#### 用于独立容器（Enterprise）

1. 确认您已在部署环境或 `.env` 文件中提供了有效的 `LANGGRAPH_CLOUD_LICENSE_KEY` 环境变量。
2. 确认密钥仍然有效，并且未超过其到期日期。