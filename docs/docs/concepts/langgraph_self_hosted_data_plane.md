---
search:
  boost: 2
---

# 自托管数据平面

自托管部署有两种版本：[自托管数据平面](./deployment_options.md#self-hosted-data-plane) 和 [自托管控制平面](./deployment_options.md#self-hosted-control-plane)。

!!! info "重要提示"

    自托管数据平面部署选项需要 [企业版](plans.md) 计划。

## 要求

- 你使用 `langgraph-cli` 和/或 [LangGraph Studio](./langgraph_studio.md) 应用在本地测试图。
- 你使用 `langgraph build` 命令来构建镜像。

## 自托管数据平面

[自托管数据平面](../cloud/deployment/self_hosted_data_plane.md) 部署选项采用“混合”模型：我们管理云中的 [控制平面](./langgraph_control_plane.md)，而你管理你云中的 [数据平面](./langgraph_data_plane.md)。此选项提供了一种安全管理数据平面基础设施的方式，同时将控制平面管理卸载给我们。使用自托管数据平面版本时，你需要使用 [LangSmith](https://smith.langchain.com/) API 密钥进行身份验证。

|                                    | [控制平面](../concepts/langgraph_control_plane.md)                                                                                 | [数据平面](../concepts/langgraph_data_plane.md)                                                                                               |
| ---------------------------------- | ------------------------------------------------------------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- |
| **它是什么？**                    | <ul><li>用于创建部署和修订的控制平面 UI</li><li>用于创建部署和修订的控制平面 API</li></ul>                               | <ul><li>用于协调部署与控制平面状态的数据平面“监听器”</li><li>LangGraph 服务器</li><li>Postgres、Redis 等</li></ul> |
| **它托管在哪里？**            | LangChain 的云                                                                                                            | 你的云                                                                                                                                          |
| **谁负责配置和管理？** | LangChain                                                                                                                 | 你                                                                                                                                                 |

有关如何部署 [LangGraph 服务器](../concepts/langgraph_server.md) 到自托管数据平面的信息，请参阅 [部署到自托管数据平面](../cloud/deployment/self_hosted_data_plane.md)。

### 架构

![Self-Hosted Data Plane Architecture](./img/self_hosted_data_plane_architecture.png)

### 计算平台

- **Kubernetes**：自托管数据平面部署选项支持将数据平面基础设施部署到任何 Kubernetes 集群。
- **Amazon ECS**：即将推出！

!!! tip
如果你想部署到 Kubernetes，可以遵循 [自托管数据平面部署指南](../cloud/deployment/self_hosted_data_plane.md)。