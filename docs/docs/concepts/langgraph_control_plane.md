---
search:
  boost: 2
---

# LangGraph 控制平面

"控制平面"一词广泛用于指代用户创建和更新 [LangGraph 服务器](./langgraph_server.md)（部署）的控制平面 UI，以及支持 UI 体验的控制平面 API。

当用户通过控制平面 UI 进行更新时，更新将存储在控制平面状态中。[LangGraph 数据平面](./langgraph_data_plane.md)“监听器”应用程序通过调用控制平面 API 来轮询这些更新。

## 控制平面 UI

通过控制平面 UI，您可以：

- 查看待处理部署的列表。
- 查看单个部署的详细信息。
- 创建新部署。
- 更新部署。
- 更新部署的环境变量。
- 查看部署的构建和服务器日志。
- 查看部署指标，如 CPU 和内存使用情况。
- 删除部署。

控制平面 UI 嵌入在 [LangSmith](https://docs.smith.langchain.com/langgraph_cloud) 中。

## 控制平面 API

本节介绍控制平面 API 的数据模型。该 API 用于创建、更新和删除部署。有关更多详细信息，请参阅 [控制平面 API 参考](../cloud/reference/api/api_ref_control_plane.md)。

### 部署

部署是 LangGraph 服务器的一个实例。一个部署可以有多个修订版本。

### 修订

修订是部署的一个迭代版本。创建新部署时，会自动创建一个初始修订版本。要部署代码更改或更新部署的密钥，必须创建新的修订版本。

## 控制平面功能

本节介绍控制平面的各种功能。

### 部署类型

为简化起见，控制平面提供两种部署类型，具有不同的资源分配：`Development` 和 `Production`。

| **部署类型** | **CPU/内存**  | **扩展**         | **数据库**                                                                     |
|-----------------|-----------------|---------------------|----------------------------------------------------------------------------------|
| Development     | 1 CPU, 1 GB RAM | 最多 1 个副本   | 10 GB 磁盘，无备份                                                              |
| Production      | 2 CPU, 2 GB RAM | 最多 10 个副本 | 自动扩展磁盘，自动备份，高可用（多区域配置）                                      |

CPU 和内存资源按副本计算。

!!! warning "不可变的部署类型"

    创建部署后，无法更改部署类型。

!!! info "自托管部署"
    [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署的资源可以完全自定义。部署类型仅适用于 [云 SaaS](../concepts/langgraph_cloud.md) 部署。

#### Production

`Production` 类型部署适用于“生产”工作负载。例如，为关键路径中的面向客户的应用程序选择 `Production`。

`Production` 类型部署的资源可以根据具体用例和容量限制，逐个手动增加。请联系 support@langchain.dev 请求增加资源。

#### Development

`Development` 类型部署适用于开发和测试。例如，为内部测试环境选择 `Development`。`Development` 类型部署不适用于“生产”工作负载。

!!! danger "可抢占的计算基础设施"
    `Development` 类型部署（API 服务器、队列服务器和数据库）在可抢占的计算基础设施上配置。这意味着计算基础设施**可能随时被终止，恕不另行通知**。这可能导致间歇性的...

    - Redis 连接超时/错误
    - Postgres 连接超时/错误
    - 失败或重试的后台运行
    
    此行为是符合预期的。可抢占的计算基础设施**大大降低了配置 `Development` 类型部署的成本**。LangGraph 服务器设计为容错的。该实现将自动尝试从 Redis/Postgres 连接错误中恢复，并重试失败的后台运行。

    `Production` 类型部署在持久计算基础设施上进行配置，而非可抢占的计算基础设施。

`Development` 类型部署的数据库磁盘大小可以根据具体用例和容量限制，逐个手动增加。对于大多数用例，应配置 [TTL](../how-tos/ttl/configure_ttl.md) 来管理磁盘使用情况。请联系 support@langchain.dev 请求增加资源。

### 数据库配置

控制平面和 [LangGraph 数据平面](./langgraph_data_plane.md) “监听器”应用程序会协调以自动为每个部署创建 Postgres 数据库。该数据库充当部署的 [持久化层](../concepts/persistence.md)。

在实现 LangGraph 应用程序时，开发人员无需配置 [checkpointer](../concepts/persistence.md#checkpointer-libraries)。而是为图自动配置 checkpointer。为图配置的任何 checkpointer 都将被自动配置的 checkpointer 替换。

无法直接访问数据库。所有数据库访问都通过 [LangGraph 服务器](../concepts/langgraph_server.md) 进行。

在删除部署之前，数据库永远不会被删除。

!!! info
    可以为 [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署配置自定义 Postgres 实例。

### 异步部署

部署和修订的基础设施是异步配置和部署的。它们不会在提交后立即部署。目前，部署最多可能需要几分钟。

- 创建新部署时，会为该部署创建一个新数据库。数据库创建是一个一次性步骤。此步骤会延长部署的初始修订版本的部署时间。
- 为部署创建后续修订版本时，没有数据库创建步骤。与初始修订版本的部署时间相比，后续修订版本的部署时间会大大缩短。
- 每个修订版本的部署过程包含一个构建步骤，可能需要几分钟时间。

控制平面和 [LangGraph 数据平面](./langgraph_data_plane.md) “监听器”应用程序会协调以实现异步部署。

### 监控

部署准备就绪后，控制平面会监控部署并记录各种指标，例如：

- 部署的 CPU 和内存使用情况。
- 容器重启次数。
- 副本数量（这将随着 [自动缩放](../concepts/langgraph_data_plane.md#autoscaling) 而增加）。
- [Postgres](../concepts/langgraph_data_plane.md#postgres) 的 CPU、内存使用情况和磁盘使用情况。
- [LangGraph 服务器队列](../concepts/langgraph_server.md#persistence-and-task-queue) 中待处理/活动的运行计数。
- [LangGraph 服务器 API](../concepts/langgraph_server.md) 的成功响应计数、错误响应计数和延迟。

这些指标在控制平面 UI 中显示为图表。

### LangSmith 集成

为每个部署自动创建一个 [LangSmith](https://docs.smith.langchain.com/) 跟踪项目。跟踪项目的名称与部署名称相同。创建部署时，无需指定 `LANGCHAIN_TRACING` 和 `LANGSMITH_API_KEY`/`LANGCHAIN_API_KEY` 环境变量；它们由控制平面自动设置。

删除部署时，跟踪和跟踪项目不会被删除。