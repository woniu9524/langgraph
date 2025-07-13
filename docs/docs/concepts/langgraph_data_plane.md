---
search:
  boost: 2
---

# LangGraph Data Plane

"Data plane" 这个术语广泛用于指代 [LangGraph Server](./langgraph_server.md)（部署）、每个服务器相应的底层基础设施，以及持续轮询 [LangGraph Control Plane](./langgraph_control_plane.md) 更新的“监听器”应用程序。

## Server Infrastructure

除了 [LangGraph Server](./langgraph_server.md) 本身之外，每个服务器的以下基础设施组件也包含在“数据平面”的广义定义中：

- Postgres
- Redis
- Secrets store
- Autoscalers

## "Listener" Application

数据平面的“监听器”应用程序会定期调用 [control plane APIs](../concepts/langgraph_control_plane.md#control-plane-api) 来：

- 确定是否应创建新部署。
- 确定是否应更新现有部署（即新版本）。
- 确定是否应删除现有部署。

换句话说，数据平面“监听器”读取控制平面的最新状态（期望状态），并采取行动来协调未完成的部署（当前状态），使其与最新状态匹配。

## Postgres

Postgres 是 LangGraph Server 中所有用户、运行和长期记忆数据的持久化层。它存储检查点（更多信息请参见 [此处](./persistence.md)）、服务器资源（线程、运行、助手和 cron）以及存储在长期记忆存储中的项（更多信息请参见 [此处](./persistence.md#memory-store)）。

## Redis

Redis 在每个 LangGraph Server 中用作服务器和队列工作程序之间进行通信的方式，并用于存储临时元数据。Redis 不存储任何用户或运行数据。

### Communication

LangGraph Server 中的所有运行都由每个部署中包含的后台工作程序池执行。为了支持这些运行的某些功能（例如取消和输出流式传输），我们需要一个通道来在服务器和处理特定运行的工作程序之间进行双向通信。我们使用 Redis 来组织这种通信。

1. Redis 列表用作在创建新运行时立即唤醒工作程序的机制。此列表中仅存储一个哨兵值，不包含实际的运行信息。运行信息随后由工作程序从 Postgres 中检索。
2. Redis 字符串和 Redis Pub/Sub 通道的组合用于服务器将运行取消请求传达给适当的工作程序。
3. 在处理运行期间，工作程序使用 Redis Pub/Sub 通道广播代理的流式输出。服务器中任何打开的 `/stream` 请求都会订阅该通道，并在事件到达时将它们转发到响应。Redis 不存储任何事件。

### Ephemeral metadata

LangGraph Server 中的运行可能会因特定故障（目前仅限于运行期间遇到的瞬态 Postgres 错误）而重试。为了限制重试次数（目前限制为每次运行 3 次尝试），我们在工作程序接收运行时将尝试次数记录在 Redis 字符串中。除了其 ID 外，这不包含任何特定于运行的信息，并在短暂延迟后过期。

## Data Plane Features

本节介绍数据平面的各种功能。

### Data Region

!!! info "仅限于 Cloud SaaS"
    数据区域仅适用于 [Cloud SaaS](../concepts/langgraph_cloud.md) 部署。

部署可以创建在 2 个数据区域：美国和欧洲。

部署的数据区域由创建部署的 LangSmith 组织的区域决定。部署及其底层数据库无法在区域之间迁移。

### Autoscaling

[`Production` 类型](../concepts/langgraph_control_plane.md#deployment-types) 部署会自动扩展到最多 10 个容器。扩展是基于 3 个指标：

1. CPU 利用率
1. 内存利用率
1. 待处理（进行中）[运行](./assistants.md#execution)的数量

对于 CPU 利用率，自动缩放器目标是 75% 的利用率。这意味着自动缩放器将向上或向下扩展容器数量，以确保 CPU 利用率接近或等于 75%。对于内存利用率，自动缩放器也以 75% 的利用率为目标。

对于待处理运行的数量，自动缩放器目标是 10 个待处理运行。例如，如果当前容器数量为 1，但待处理运行数为 20，自动缩放器将把部署扩展到 2 个容器（20 个待处理运行 / 2 个容器 = 每个容器 10 个待处理运行）。

每个指标都是独立计算的，自动缩放器将根据导致最多容器的指标来确定缩放操作。

缩减操作会延迟 30 分钟后才会执行。换句话说，如果自动缩放器决定缩减部署，它将首先等待 30 分钟，然后才缩减。30 分钟后，将重新计算指标，如果重新计算后的指标导致容器数量少于当前数量，则部署将缩减。否则，部署将保持扩展状态。此“冷却”期可确保部署不会过于频繁地向上和向下扩展。

### Static IP Addresses

!!! info "仅限于 Cloud SaaS"
    静态 IP 地址仅适用于 [Cloud SaaS](../concepts/langgraph_cloud.md) 部署。

2025 年 1 月 6 日之后创建的所有部署的流量都将通过 NAT 网关。此 NAT 网关将拥有几个静态 IP 地址，具体取决于数据区域。有关静态 IP 地址列表，请参阅下表：

| US             | EU             |
|----------------|----------------|
| 35.197.29.146  | 34.13.192.67   |
| 34.145.102.123 | 34.147.105.64  |
| 34.169.45.153  | 34.90.22.166   |
| 34.82.222.17   | 34.147.36.213  |
| 35.227.171.135 | 34.32.137.113  | 
| 34.169.88.30   | 34.91.238.184  |
| 34.19.93.202   | 35.204.101.241 |
| 34.19.34.50    | 35.204.48.32   |

### Custom Postgres

!!! info 
    自定义 Postgres 实例仅适用于 [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_data_plane.md) 和 [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可以使用自定义 Postgres 实例代替 [由控制平面自动创建的一个](./langgraph_control_plane.md#database-provisioning)。指定 [`POSTGRES_URI_CUSTOM`](../cloud/reference/env_var.md#postgres_uri_custom) 环境变量即可使用自定义 Postgres 实例。

多个部署可以共享同一个 Postgres 实例。例如，对于 `Deployment A`，可以设置 `POSTGRES_URI_CUSTOM` 为 `postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>`，对于 `Deployment B`，可以设置 `POSTGRES_URI_CUSTOM` 为 `postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`。`<database_name_1>` 和 `database_name_2` 是同一实例内的不同数据库，但是 `<hostname_1>` 是共享的。**不能将同一个数据库用于独立的部署**。

### Custom Redis

!!! info
    自定义 Redis 实例仅适用于 [Self-Hosted Data Plane](../concepts/langgraph_self_hosted_control_plane.md) 和 [Self-Hosted Control Plane](../concepts/langgraph_self_hosted_control_plane.md) 部署。

可以使用自定义 Redis 实例代替由控制平面自动创建的一个实例。指定 [REDIS_URI_CUSTOM](../cloud/reference/env_var.md#redis_uri_custom) 环境变量即可使用自定义 Redis 实例。


多个部署可以共享同一个 Redis 实例。例如，对于 `Deployment A`，可以设置 `REDIS_URI_CUSTOM` 为 `redis://<hostname_1>:<port>/1`，对于 `Deployment B`，可以设置 `REDIS_URI_CUSTOM` 为 `redis://<hostname_1>:<port>/2`。`1` 和 `2` 是同一实例内的不同数据库编号，但是 `<hostname_1>` 是共享的。**不能将同一个数据库编号用于独立的部署**。

### LangSmith Tracing

LangGraph Server 已自动配置为将 tracing 发送到 LangSmith。有关每个部署选项的详细信息，请参阅下表。

| Cloud SaaS | Self-Hosted Data Plane | Self-Hosted Control Plane | Standalone Container |
|------------|------------------------|---------------------------|----------------------|
| Required<br><br>Trace to LangSmith SaaS. | Optional<br><br>Disable tracing or trace to LangSmith SaaS. | Optional<br><br>Disable tracing or trace to Self-Hosted LangSmith. | Optional<br><br>Disable tracing, trace to LangSmith SaaS, or trace to Self-Hosted LangSmith. |

### Telemetry

LangGraph Server 已自动配置为发送遥测元数据以用于计费目的。有关每个部署选项的详细信息，请参阅下表。

| Cloud SaaS | Self-Hosted Data Plane | Self-Hosted Control Plane | Standalone Container |
|------------|------------------------|---------------------------|----------------------|
| Telemetry sent to LangSmith SaaS. | Telemetry sent to LangSmith SaaS. | Self-reported usage (audit) for air-gapped license key.<br><br>Telemetry sent to LangSmith SaaS for LangGraph Platform License Key. | Self-reported usage (audit) for air-gapped license key.<br><br>Telemetry sent to LangSmith SaaS for LangGraph Platform License Key. |

### Licensing

LangGraph Server 已自动配置为执行许可证密钥验证。有关每个部署选项的详细信息，请参阅下表。

| Cloud SaaS | Self-Hosted Data Plane | Self-Hosted Control Plane | Standalone Container |
|------------|------------------------|---------------------------|----------------------|
| LangSmith API Key validated against LangSmith SaaS. | LangSmith API Key validated against LangSmith SaaS. | Air-gapped license key or LangGraph Platform License Key validated against LangSmith SaaS. | Air-gapped license key or LangGraph Platform License Key validated against LangSmith SaaS. |