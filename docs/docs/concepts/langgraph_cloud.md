---
search:
  boost: 2
---

# Cloud SaaS

要部署 [LangGraph Server](../concepts/langgraph_server.md)，请遵循 [如何部署到 Cloud SaaS](../cloud/deployment/cloud.md) 的操作指南。

## 概述

Cloud SaaS 部署选项是一种完全托管的部署模式，其中 LangChainCloud 管理 [控制平面](./langgraph_control_plane.md) 和 [数据平面](./langgraph_data_plane.md)。

|                                    | [控制平面](../concepts/langgraph_control_plane.md)                                                                                                                   | [数据平面](../concepts/langgraph_data_plane.md)                                                                                                        |
| ---------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------- |
| **它是什么？**                    | <ul><li>用于创建部署和版本的控制平面 UI</li><li>用于创建部署和版本的控制平面 API</li></ul>                                                                                           | <ul><li>用于协调部署与控制平面状态的数据平面“监听器”</li><li>LangGraph Servers</li><li>Postgres、Redis 等</li></ul>                                                                                |
| **在哪里托管？**            | LangChain Cloud                                                                                                                                          | LangChain Cloud                                                                                                                                    |
| **谁负责预配和管理？** | LangChain                                                                                                                                                       | LangChain                                                                                                                                          |

## 架构

![Cloud SaaS](./img/self_hosted_control_plane_architecture.png)