---
search:
  boost: 2
---

# 部署选项

## 免费部署

通过 LangGraph Server 进行 LangGraph 应用部署有两种免费选项：

1. [本地部署](../tutorials/langgraph-platform/local-server.md)：用于本地测试和开发。
1. [独立容器（精简版）](../concepts/langgraph_standalone_container.md)：独立容器的精简版本，适用于每年节点执行次数可能不会超过 100 万次且不需要 Cron 和其他企业级功能的部署。独立容器（精简版）部署选项免费提供，需提供 LangSmith API 密钥。

## 生产部署

通过 [LangGraph Platform](langgraph_platform.md) 进行部署有 4 种主要选项：

1. [Cloud SaaS](#cloud-saas)

1. [自托管数据平面](#self-hosted-data-plane)

1. [自托管控制平面](#self-hosted-control-plane)

1. [独立容器](#standalone-container)


快速对比：

|                      | **Cloud SaaS** | **自托管数据平面** | **自托管控制平面** | **独立容器** |
|----------------------|----------------|----------------------------|-------------------------------|--------------------------|
| **[控制平面 UI/API](../concepts/langgraph_control_plane.md)** | 是 | 是 | 是 | 否 |
| **CI/CD** | 由平台内部管理 | 由您外部管理 | 由您外部管理 | 由您外部管理 |
| **数据/计算驻留** | LangChain 的云 | 您的云 | 您的云 | 您的云 |
| **LangSmith 兼容性** | 追踪到 LangSmith SaaS | 追踪到 LangSmith SaaS | 追踪到自托管 LangSmith | 可选追踪 |
| **[服务器版本兼容性](../concepts/langgraph_server.md#server-versions)** | 企业版 | 企业版 | 企业版 | 精简版、企业版 |
| **[定价](https://www.langchain.com/pricing-langgraph-platform)** | Plus | 企业版 | 企业版 | 开发者版 |

## Cloud SaaS

[Cloud SaaS](./langgraph_cloud.md) 部署选项是一种完全托管的部署模式，我们在云端管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项提供了一种简单的方式来部署和管理您的 LangGraph Server。

将您的 GitHub 存储库连接到平台，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph Server。构建过程（即 CI/CD）由平台内部管理。

更多信息，请参阅：

* [Cloud SaaS 概念指南](./langgraph_cloud.md)
* [如何部署到 Cloud SaaS](../cloud/deployment/cloud.md)

## 自托管数据平面

!!! info "重要提示"
    自托管数据平面部署选项需要 [企业版](../concepts/plans.md) 计划。

[自托管数据平面](./langgraph_self_hosted_data_plane.md) 部署选项是一种“混合”部署模式，我们在云端管理 [控制平面](./langgraph_control_plane.md)，您在自己的云中管理 [数据平面](./langgraph_data_plane.md)。此选项提供了一种安全管理数据平面基础设施的方式，同时将控制平面管理外包给我们。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 镜像，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph Server。

支持的计算平台：[Kubernetes](https://kubernetes.io/)、[Amazon ECS](https://aws.amazon.com/ecs/)（即将推出！）

更多信息，请参阅：

* [自托管数据平面概念指南](./langgraph_self_hosted_data_plane.md)
* [如何部署自托管数据平面](../cloud/deployment/self_hosted_data_plane.md)

## 自托管控制平面

!!! info "重要提示"
    自托管控制平面部署选项需要 [企业版](../concepts/plans.md) 计划。

[自托管控制平面](./langgraph_self_hosted_control_plane.md) 部署选项是一种完全自托管的部署模式，您在自己的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项让您完全掌控并负责控制平面和数据平面基础设施。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 镜像，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph Server。

支持的计算平台：[Kubernetes](https://kubernetes.io/)

更多信息，请参阅：

* [自托管控制平面概念指南](./langgraph_self_hosted_control_plane.md)
* [如何部署自托管控制平面](../cloud/deployment/self_hosted_control_plane.md)

## 独立容器

[独立容器](./langgraph_standalone_container.md) 部署选项是最不具限制性的部署模式。在您的云中部署 LangGraph Server 的独立实例，使用任何 [可用](./plans.md) 的许可证选项。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 镜像，并使用您选择的容器部署工具部署您的 LangGraph Server。镜像可以部署到任何计算平台。

更多信息，请参阅：

* [独立容器概念指南](./langgraph_standalone_container.md)
* [如何部署独立容器](../cloud/deployment/standalone_container.md)

## 相关

更多信息，请参阅：

* [LangGraph Platform 计划](./plans.md)
* [LangGraph Platform 定价](https://www.langchain.com/langgraph-platform-pricing)