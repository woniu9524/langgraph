---
search:
  boost: 2
---

# 单独的容器

要部署 [LangGraph Server](../concepts/langgraph_server.md)，请按照 [如何部署单独的容器](../cloud/deployment/standalone_container.md) 的操作指南进行。

## 概述

单独的容器部署选项是对部署限制最少的模型。没有 [控制平面](./langgraph_control_plane.md)。[数据平面](./langgraph_data_plane.md) 基础架构由您管理。

|                   | [控制平面](../concepts/langgraph_control_plane.md) | [数据平面](../concepts/langgraph_data_plane.md) |
|-------------------|-------------------|------------|
| **是什么？** | 不适用 | <ul><li>LangGraph Servers</li><li>Postgres, Redis 等</li></ul> |
| **托管在哪里？** | 不适用 | 您的云 |
| **由谁提供和管理？** | 不适用 | 您 |

!!! warning

      LangGraph Platform 不应在无服务器环境中部署。缩减到零可能会导致任务丢失，并且向上扩展将无法可靠运行。

## 架构

![Standalone Container](./img/langgraph_platform_deployment_architecture.png)

## 计算平台

### Kubernetes

单独的容器部署选项支持将数据平面基础架构部署到 Kubernetes 集群。

### Docker

单独的容器部署选项支持将数据平面基础架构部署到任何受 Docker 支持的计算平台。

## Lite 与 Enterprise 版

单独的容器部署选项支持两种 [服务器版本](../concepts/langgraph_server.md#langgraph-server)：

- `Lite` 版本是免费的，但功能有限。
- `Enterprise` 版本具有自定义定价，功能齐全。

有关功能差异的更多详细信息，请参阅 [LangGraph Server](../concepts/langgraph_server.md#server-versions)。