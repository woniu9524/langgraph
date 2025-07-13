# 自托管控制平面

自托管部署有两种版本：[自托管数据平面](./deployment_options.md#self-hosted-data-plane) 和 [自托管控制平面](./deployment_options.md#self-hosted-control-plane)。

!!! info "重要"
    自托管控制平面部署选项需要 [企业版](plans.md) 计划。

## 要求

- 您使用 `langgraph-cli` 和/或 [LangGraph Studio](./langgraph_studio.md) 应用在本地测试图。
- 您使用 `langgraph build` 命令构建镜像。
- 您已部署自托管 LangSmith 实例。
- 您正在为您的 LangSmith 实例使用 Ingress。所有代理将作为此 Ingress 后面的 Kubernetes 服务进行部署。

## 自托管控制平面

[自托管控制平面](./langgraph_self_hosted_control_plane.md) 部署选项是一种完全自托管的部署模式，您可以在云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项赋予您对控制平面和数据平面基础设施的完全控制和责任。

|                   | [控制平面](../concepts/langgraph_control_plane.md) | [数据平面](../concepts/langgraph_data_plane.md) |
|-------------------|-------------------|------------|
| **它是什么？** | <ul><li>用于创建部署和修订的控制平面 UI</li><li>用于创建部署和修订的控制平面 API</li></ul> | <ul><li>用于协调部署与控制平面状态的数据平面“监听器”</li><li>LangGraph 服务器</li><li>Postgres、Redis 等</li></ul> |
| **它托管在哪里？** | 您的云 | 您的云 |
| **谁负责配置和管理？** | 您 | 您 |

### 架构

![自托管控制平面架构](./img/self_hosted_control_plane_architecture.png)

### 计算平台

 - **Kubernetes**：自托管控制平面部署选项支持将控制平面和数据平面基础设施部署到任何 Kubernetes 集群。

!!! tip
    如果您想在您的 LangSmith 实例上启用此功能，请遵循 [自托管控制平面部署指南](../cloud/deployment/self_hosted_control_plane.md)。