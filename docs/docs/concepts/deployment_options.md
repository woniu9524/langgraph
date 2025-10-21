---
search:
  boost: 2
---

# 部署选项

## 免费部署

[本地](../tutorials/langgraph-platform/local-server.md)：用于本地测试和开发部署。

## 生产部署

在 [LangGraph Platform](langgraph_platform.md) 上部署有 4 种主要选项：

1. [云 SaaS](#cloud-saas)

1. [自托管数据平面](#self-hosted-data-plane)

1. [自托管控制平面](#self-hosted-control-plane)

1. [独立容器](#standalone-container)


快速对比：

|                      | **云 SaaS** | **自托管数据平面** | **自托管控制平面** | **独立容器** |
|----------------------|----------------|----------------------------|-------------------------------|--------------------------|
| **[控制平面 UI/API](../concepts/langgraph_control_plane.md）** | 是 | 是 | 是 | 否 |
| **CI/CD** | 由平台内部管理 | 由您外部管理 | 由您外部管理 | 由您外部管理 |
| **数据/计算驻留** | LangChain 的云 | 您的云 | 您的云 | 您的云 |
| **LangSmith 兼容性** | 追踪到 LangSmith SaaS | 追踪到 LangSmith SaaS | 追踪到自托管 LangSmith | 可选追踪 |
| **[定价](https://www.langchain.com/pricing-langgraph-platform)** | Plus | Enterprise | Enterprise | Enterprise |

## 云 SaaS

[云 SaaS](./langgraph_cloud.md) 部署选项是一种完全托管的部署模式，我们将在我们的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项提供了一种简单的方式来部署和管理您的 LangGraph 服务器。

将您的 GitHub 仓库连接到平台，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph 服务器。构建过程（即 CI/CD）由平台内部管理。

更多信息，请参阅：

* [云 SaaS 概念指南](./langgraph_cloud.md)
* [如何部署到云 SaaS](../cloud/deployment/cloud.md)

## 自托管数据平面

!!! info "重要信息"
    自托管数据平面部署选项需要 [Enterprise](../concepts/plans.md) 计划。

[自托管数据平面](./langgraph_self_hosted_data_plane.md) 部署选项是一种“混合”部署模式，我们将在我们的云中管理 [控制平面](./langgraph_control_plane.md)，而您将在您的云中管理 [数据平面](./langgraph_data_plane.md)。此选项提供了一种安全管理您的数据平面基础设施的方式，同时将控制平面的管理外包给我们。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 镜像，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph 服务器。

支持的计算平台：[Kubernetes](https://kubernetes.io/)、[Amazon ECS](https://aws.amazon.com/ecs/)（即将推出！）

更多信息，请参阅：

* [自托管数据平面概念指南](./langgraph_self_hosted_data_plane.md)
* [如何部署自托管数据平面](../cloud/deployment/self_hosted_data_plane.md)

## 自托管控制平面

!!! info "重要信息"
    自托管控制平面部署选项需要 [Enterprise](../concepts/plans.md) 计划。

[自托管控制平面](./langgraph_self_hosted_control_plane.md) 部署选项是一种完全自托管的部署模式，您将在您的云中管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。此选项使您能够完全控制和负责控制平面和数据平面基础设施。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 镜像，并从 [控制平面 UI](./langgraph_control_plane.md#control-plane-ui) 部署您的 LangGraph 服务器。

支持的计算平台：[Kubernetes](https://kubernetes.io/)

更多信息，请参阅：

* [自托管控制平面概念指南](./langgraph_self_hosted_control_plane.md)
* [如何部署自托管控制平面](../cloud/deployment/self_hosted_control_plane.md)

## 独立容器

[独立容器](./langgraph_standalone_container.md) 部署选项是最不具限制性的部署模式。使用您云中的 LangGraph 服务器独立实例进行部署，并使用任何 [可用](./plans.md) 的许可选项。

使用 [LangGraph CLI](./langgraph_cli.md) 构建 Docker 镜像，并使用您选择的容器部署工具来部署您的 LangGraph 服务器。镜像可以部署到任何计算平台。

更多信息，请参阅：

* [独立容器概念指南](./langgraph_standalone_container.md)
* [如何部署独立容器](../cloud/deployment/standalone_container.md)

## 相关

更多信息，请参阅：

* [LangGraph Platform 计划](./plans.md)
* [LangGraph Platform 定价](https://www.langchain.com/langgraph-platform-pricing)