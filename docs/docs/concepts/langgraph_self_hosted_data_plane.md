---
search:
  boost: 2
---

# 自托管数据平面

自托管部署有两种版本：[自托管数据平面](./deployment_options.md#self-hosted-data-plane) 和 [自托管控制平面](./deployment_options.md#self-hosted-control-plane)。

!!! info "重要提示"
    自托管数据平面部署选项需要企业版 [Enterprise](plans.md)。

## 要求

- 您将使用 `langgraph-cli` 和/或 [LangGraph Studio](./langgraph_studio.md) 应用在本地测试图。
- 您将使用 `langgraph build` 命令来构建镜像。

## 自托管数据平面

[自托管数据平面](../cloud/deployment/self_hosted_data_plane.md) 部署选项是一种“混合”部署模式，其中我们管理云中的 [控制平面](./langgraph_control_plane.md)，而您管理云中的 [数据平面](./langgraph_data_plane.md)。此选项提供了一种安全管理数据平面基础设施的方法，同时将控制平面管理委托给我们。在使用自托管数据平面版本时，您将使用 [LangSmith](https://smith.langchain.com/) API 密钥进行身份验证。

|                   | [控制平面](../concepts/langgraph_control_plane.md) | [数据平面](../concepts/langgraph_data_plane.md) |
|-------------------|---------------------------------------------------|---------------------------------------------------|
| **是什么？** | <ul><li>用于创建部署和修订的控制平面 UI</li><li>用于创建部署和修订的控制平面 API</li></ul> | <ul><li>用于将部署与控制平面状态进行协调的数据平面“监听器”</li><li>LangGraph 服务器</li><li>Postgres, Redis 等</li></ul> |
| **托管在哪里？** | LangChain 的云                                    | 您的云                                            |
| **谁负责配置和管理？** | LangChain                                         | 您                                                |

有关如何将 [LangGraph 服务器](../concepts/langgraph_server.md) 部署到自托管数据平面的信息，请参阅 [部署到自托管数据平面](../cloud/deployment/self_hosted_data_plane.md)。

### 架构

![自托管数据平面架构](./img/self_hosted_data_plane_architecture.png)

### 计算平台

- **Kubernetes**: 自托管数据平面部署选项支持将数据平面基础设施部署到任何 Kubernetes 集群。
- **Amazon ECS**: 即将推出！

!!! tip
    如果您想部署到 Kubernetes，可以遵循 [自托管数据平面部署指南](../cloud/deployment/self_hosted_data_plane.md)。