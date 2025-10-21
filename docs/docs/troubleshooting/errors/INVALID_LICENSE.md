# 无效许可证

在尝试启动自托管 LangGraph Platform 服务器时，许可证验证失败会引发此错误。此错误特定于 LangGraph Platform，与开源库无关。

## 发生情况

在没有有效企业许可证或 API 密钥的情况下运行 LangGraph Platform 的自托管部署时，会发生此错误。

## 故障排除

### 确认部署类型

首先，请确认所需的部署模式。

#### 用于本地开发

如果只是在本地进行开发，可以通过运行 `langgraph dev` 来使用轻量级的内存服务器。
有关更多信息，请参阅 [本地服务器](../../tutorials/langgraph-platform/local-server.md) 文档。

#### 用于托管 LangGraph Platform

如果您需要一个快速的托管环境，可以考虑 [SaaS 云部署](../../concepts/langgraph_cloud.md) 选项。此选项不需要额外的许可证密钥。

#### 用于独立容器

对于自托管，请设置 `LANGGRAPH_CLOUD_LICENSE_KEY` 环境变量。如果您对企业许可证密钥感兴趣，请联系 LangChain 支持团队。

有关部署选项及其功能的更多信息，请参阅 [部署选项](../../concepts/deployment_options.md) 文档。

### 确认凭据

如果您已确认要自托管 LangGraph Platform，请验证您的凭据。

#### 用于独立容器

1. 确认您已在部署环境或 `.env` 文件中提供了有效的 `LANGGRAPH_CLOUD_LICENSE_KEY` 环境变量。
2. 确认密钥仍然有效且未超出其有效期。