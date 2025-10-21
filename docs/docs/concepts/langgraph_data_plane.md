---
search:
  boost: 2
---

# LangGraph 数据平面

“数据平面”一词广泛用于指代 [LangGraph 服务器](./langgraph_server.md)（部署）、每个服务器的基础设施以及负责持续轮询 [LangGraph 控制平面](./langgraph_control_plane.md)更新的“监听器”应用程序。

## 服务器基础架构

除了 [LangGraph 服务器](./langgraph_server.md)本身，每个服务器的以下基础设施组件也包含在“数据平面”的广义定义中：

- Postgres
- Redis
- Secrets store
- Autoscalers

## “监听器”应用程序

数据平面“监听器”应用程序会定期调用 [控制平面 API](../concepts/langgraph_control_plane.md#control-plane-api)，用于：

- 确定是否应创建新部署。
- 确定是否应更新现有部署（即新版本）。
- 确定是否应删除现有部署。

换句话说，数据平面“监听器”读取控制平面的最新状态（期望状态），并采取行动来协调待处理的部署（当前状态），使其与最新状态匹配。

## Postgres

Postgres 是 LangGraph 服务器中所有用户、运行和长期内存数据的持久化层。它存储了检查点（更多信息请参见 [此处](./persistence.md)）、服务器资源（线程、运行、助手和 cron）以及存储在长期内存存储中的项目（更多信息请参见 [此处](./persistence.md#memory-store)）。

## Redis

Redis 在每个 LangGraph 服务器中用作服务器和队列工作者之间通信的媒介，并用于存储临时元数据。Redis 中不存储任何用户或运行数据。

### 通信

LangGraph 服务器中的所有运行都由每个部署中包含的后台工作者池执行。为了启用这些运行的某些功能（例如取消和输出流式传输），我们需要一个通道来进行服务器与处理特定运行的工作者之间的双向通信。我们使用 Redis 来组织此通信。

1.  Redis 列表用作一种机制，可在创建新运行时立即唤醒工作者。此列表中仅存储哨兵值，不存储实际的运行信息。运行信息随后由工作者从 Postgres 中检索。
2.  Redis 字符串和 Redis PubSub 通道的组合用于服务器将运行取消请求传达给相应的工作者。
3.  Redis PubSub 通道由工作者在处理运行时广播来自代理的流式输出。服务器中任何打开的 `/stream` 请求都会订阅该通道，并在事件到达时将其转发给响应。Redis 中不会存储任何事件。

### 临时元数据

LangGraph 服务器中的运行可能会针对特定故障（目前仅针对运行过程中遇到的瞬态 Postgres 错误）进行重试。为了限制重试次数（目前每次运行最多 3 次），当我们在 Redis 字符串中记录尝试次数时，它会被拾取。除了其 ID 外，它不包含任何特定于运行的信息，并且在短暂延迟后会过期。

## 数据平面功能

本节介绍数据平面的各种功能。

### 数据区域

!!! info "仅适用于云 SaaS"
    数据区域仅适用于 [云 SaaS](./langgraph_cloud.md) 部署。

部署可以创建在 2 个数据区域：美国（US）和欧洲（EU）。

部署的数据区域由创建部署的 LangSmith 组织的数据区域隐含。部署及其底层数据库不能在数据区域之间迁移。

### 自动缩放

[`Production` 类型](../concepts/langgraph_control_plane.md#deployment-types)部署会自动扩展到 10 个容器。扩展基于 3 个指标：

1.  CPU 利用率
1.  内存利用率
1.  待处理（进行中）[运行](./assistants.md#execution)的数量

对于 CPU 利用率，自动缩放器以 75% 的利用率为目标。这意味着自动缩放器会向上或向下扩展容器数量，以确保 CPU 利用率达到或接近 75%。对于内存利用率，也以 75% 的利用率为目标。

对于待处理运行的数量，自动缩放器以 10 个待处理运行为目标。例如，如果当前容器数量为 1，但待处理运行数为 20，则自动缩放器会将部署扩展到 2 个容器（20 个待处理运行/2 个容器 = 每个容器 10 个待处理运行）。

每个指标独立计算，自动缩放器将根据导致最多容器的指标来确定缩放操作。

向下缩放操作会延迟 30 分钟，然后才会执行。换句话说，如果自动缩放器决定向下缩放部署，它将首先等待 30 分钟，然后再进行缩放。30 分钟后，将重新计算指标，如果重新计算的指标导致容器数量少于当前数量，则部署将向下缩放。否则，部署将保持扩展状态。此“冷却”期可确保部署不会过于频繁地进行扩展和缩减。

### 静态 IP 地址

!!! info "仅适用于云 SaaS"
    静态 IP 地址仅适用于 [云 SaaS](./langgraph_cloud.md) 部署。

自 2025 年 1 月 6 日起创建的部署的所有流量都将通过 NAT 网关。此 NAT 网关将具有几个静态 IP 地址，具体取决于数据区域。有关静态 IP 地址列表，请参阅下表：

| US             | EU             |
| -------------- | -------------- |
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.46  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  |
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |

### 自定义 Postgres

!!! info
自定义 Postgres 实例仅适用于 [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可以使用自定义 Postgres 实例，而不是 [由控制平面自动创建的实例](./langgraph_control_plane.md#database-provisioning)。通过指定 [`POSTGRES_URI_CUSTOM`](../cloud/reference/env_var.md#postgres_uri_custom) 环境变量来使用自定义 Postgres 实例。

多个部署可以共享同一个 Postgres 实例。例如，对于 `Deployment A`，可以将 `POSTGRES_URI_CUSTOM` 设置为 `postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>`，对于 `Deployment B`，可以将 `POSTGRES_URI_CUSTOM` 设置为 `postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`。`<database_name_1>` 和 `<database_name_2>` 是同一实例内的不同数据库，但 `<hostname_1>` 是共享的。**不能为单独的部署使用相同的数据库**。

### 自定义 Redis

!!! info
自定义 Redis 实例仅适用于 [自托管数据平面](../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可以使用自定义 Redis 实例，而不是由控制平面自动创建的实例。通过指定 [REDIS_URI_CUSTOM](../cloud/reference/env_var.md#redis_uri_custom) 环境变量来使用自定义 Redis 实例。

多个部署可以共享同一个 Redis 实例。例如，对于 `Deployment A`，可以将 `REDIS_URI_CUSTOM` 设置为 `redis://<hostname_1>:<port>/1`，对于 `Deployment B`，可以将 `REDIS_URI_CUSTOM` 设置为 `redis://<hostname_1>:<port>/2`。`1` 和 `2` 是同一实例内的不同数据库编号，但 `<hostname_1>` 是共享的。**不能为单独的部署使用相同的数据库编号**。

### LangSmith 跟踪

LangGraph 服务器已自动配置为将跟踪发送到 LangSmith。有关各种部署选项的详细信息，请参阅下表。

| Cloud SaaS                               | Self-Hosted Data Plane                                      | Self-Hosted Control Plane                                          | Standalone container                                                                         |
| ---------------------------------------- | ----------------------------------------------------------- | ------------------------------------------------------------------ | -------------------------------------------------------------------------------------------- |
| Required<br><br>Trace to LangSmith SaaS. | Optional<br><br>Disable tracing or trace to LangSmith SaaS. | Optional<br><br>Disable tracing or trace to Self-Hosted LangSmith. | Optional<br><br>Disable tracing, trace to LangSmith SaaS, or trace to Self-Hosted LangSmith. |

### 遥测

LangGraph 服务器已自动配置为报告遥测元数据，用于计费目的。有关各种部署选项的详细信息，请参阅下表。

| Cloud SaaS                        | Self-Hosted Data Plane            | Self-Hosted Control Plane                                                                                                                             | Standalone container                                                                                                                                        |
| --------------------------------- | --------------------------------- | ---------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------- |
| Telemetry sent to LangSmith SaaS. | Telemetry sent to LangSmith SaaS. | Self-reported usage (audit) for air-gapped license key.<br><br>Telemetry sent to LangSmith SaaS for LangGraph Platform License Key. | Self-reported usage (audit) for air-gapped license key.<br><br>Telemetry sent to LangSmith SaaS for LangGraph Platform License Key. |

### 许可

LangGraph 服务器已自动配置为执行许可证密钥验证。有关各种部署选项的详细信息，请参阅下表。

| Cloud SaaS                                          | Self-Hosted Data Plane                              | Self-Hosted Control Plane                                                                  | Standalone container                                                                       |
| --------------------------------------------------- | --------------------------------------------------- | ------------------------------------------------------------------------------------------ | ------------------------------------------------------------------------------------------ |
| LangSmith API Key validated against LangSmith SaaS. | LangSmith API Key validated against LangSmith SaaS. | Air-gapped license key or LangGraph Platform License Key validated against LangSmith SaaS. | Air-gapped license key or LangGraph Platform License Key validated against LangSmith SaaS. |