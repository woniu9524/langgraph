# 环境变量

LangGraph Server 支持以下环境变量进行部署配置。

## `BG_JOB_ISOLATED_LOOPS`

将 `BG_JOB_ISOLATED_LOOPS` 设置为 `True`，可将后台运行任务放置在与服务 API 事件循环隔离的独立事件循环中执行。

如果图/节点实现中包含同步代码，则此环境变量应设置为 `True`。在这种情况下，同步代码会阻塞服务 API 事件循环，可能导致 API 不可用。API 不可用的迹象是由于健康检查失败而导致的持续应用程序重启。

默认为 `False`。

## `BG_JOB_SHUTDOWN_GRACE_PERIOD_SECS`

指定服务器在队列收到关闭信号后等待后台作业完成的时间（以秒为单位）。在此期限之后，服务器将强制终止。默认为 `180` 秒。设置此项可确保作业在关闭期间有足够的时间干净地完成。添加于 `langgraph-api==0.2.16`。

## `BG_JOB_TIMEOUT_SECS`

可以增加后台运行的超时时间。但是，Cloud SaaS 部署的基础设施对 API 请求强制执行 1 小时的超时限制。这意味着客户端和服务器之间的连接将在 1 小时后超时。此设置不可配置。

后台运行可以执行超过 1 小时，但客户端必须重新连接到服务器（例如，通过 `POST /threads/{thread_id}/runs/{run_id}/stream` 加入流）才能在运行时间超过 1 小时的情况下检索输出。

默认为 `3600`。

## `DD_API_KEY`

指定 `DD_API_KEY`（您的 [Datadog API 密钥](https://docs.datadoghq.com/account_management/api-app-keys/)）可自动为部署启用 Datadog 追踪。指定其他 [`DD_*` 环境变量](https://ddtrace.readthedocs.io/en/stable/configuration.html)可配置追踪 instrumention。

如果指定了 `DD_API_KEY`，应用程序进程将被包装在 [`ddtrace-run` 命令](https://ddtrace.readthedocs.io/en/stable/installation_quickstart.html)中。通常需要其他 `DD_*` 环境变量（例如 `DD_SITE`、`DD_ENV`、`DD_SERVICE`、`DD_TRACE_ENABLED`）来正确配置追踪 instrumention。有关更多详细信息，请参阅 [`DD_*` 环境变量](https://ddtrace.readthedocs.io/en/stable/configuration.html)。

!!! note 
    启用 `DD_API_KEY`（以及 `ddtrace-run`）可能会覆盖或干扰您已 instrumented 到应用程序代码中的其他自动 instrumention 解决方案（如 OpenTelemetry）。

## `LANGCHAIN_TRACING_SAMPLING_RATE`

发送到 LangSmith 的追踪的采样率。有效值：介于 `0` 和 `1` 之间的任意浮点数。

有关更多详细信息，请参阅 <a href="https://docs.smith.langchain.com/how_to_guides/tracing/sample_traces" target="_blank">LangSmith 文档</a>。

## `LANGGRAPH_AUTH_TYPE`

LangGraph Server 部署的认证类型。有效值：`langsmith`、`noop`。

部署到 LangGraph Platform 时，此环境变量会自动设置。对于本地开发或身份验证由外部处理的部署（例如自托管），请将此环境变量设置为 `noop`。

## `LANGGRAPH_POSTGRES_POOL_MAX_SIZE`

从 langgraph-api 版本 `0.2.12` 开始，可以通过 `LANGGRAPH_POSTGRES_POOL_MAX_SIZE` 环境变量控制 Postgres 连接池的最大大小（每个副本）。通过设置此变量，您可以确定服务器将与 Postgreslush 数据库建立的并发连接的上限。

例如，如果一个部署扩展到 10 个副本，并且 `LANGGRAPH_POSTGRES_POOL_MAX_SIZE` 配置为 `150`，那么最多可以建立 `1500` 个到 Postgres 的连接。这对于数据库资源有限（或更可用）的部署，或者需要调整连接行为以提高性能或扩展性的部署特别有用。

默认为 `150` 个连接。

## `LANGSMITH_RUNS_ENDPOINTS`

仅适用于具有 [自托管 LangSmith](https://docs.smith.langchain.com/self_hosting) 的部署。

设置此环境变量可使部署将追踪发送到自托管的 LangSmith 实例。`LANGSMITH_RUNS_ENDPOINTS` 的值是 JSON 字符串：`{"<SELF_HOSTED_LANGSMITH_HOSTNAME>":"<LANGSMITH_API_KEY>"}`。

`SELF_HOSTED_LANGSMITH_HOSTNAME` 是自托管 LangSmith 实例的主机名。部署必须能够访问它。`LANGSMITH_API_KEY` 是从自托管 LangSmith 实例生成的 LangSmith API 密钥。

## `LANGSMITH_TRACING`

将 `LANGSMITH_TRACING` 设置为 `false` 可禁用到 LangSmith 的追踪。

默认为 `true`。

## `LOG_COLOR`

这主要在通过 `langgraph dev` 命令使用开发服务器的上下文中相关。将 `LOG_COLOR` 设置为 `true` 可在默认控制台渲染器中使用时启用 ANSI 颜色的控制台输出。通过将此变量设置为 `false` 来禁用颜色输出会产生单色日志。默认为 `true`。

## `LOG_LEVEL`

配置 [日志级别](https://docs.python.org/3/library/logging.html#logging-levels)。默认为 `INFO`。

## `LOG_JSON`

将 `LOG_JSON` 设置为 `true` 可使用配置的 `JSONRenderer` 将所有日志消息渲染为 JSON 对象。这会生成结构化日志，可被日志管理系统轻松解析或摄取。默认为 `false`。

## `MOUNT_PREFIX`

!!! info "仅允许用于自托管部署"
    `MOUNT_PREFIX` 环境变量仅允许用于自托管部署模型，LangGraph Platform SaaS 不允许使用此环境变量。

设置 `MOUNT_PREFIX` 可在特定路径前缀下提供 LangGraph Server。如果您部署的服务器位于需要特定路径前缀的反向代理或负载均衡器后面，这会很有用。

例如，如果服务器要在 `https://example.com/langgraph` 下提供服务，请将 `MOUNT_PREFIX` 设置为 `/langgraph`。

## `N_JOBS_PER_WORKER`

LangGraph Server 任务队列的每个 worker 的作业数。默认为 `10`。

## `POSTGRES_URI_CUSTOM`

!!! info "仅适用于自托管数据平面和自托管控制平面"
    自定义 Postgres 实例仅适用于 [自托管数据平面](../../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../../concepts/langgraph_self_hosted_control_plane.md) 部署。

指定 `POSTGRES_URI_CUSTOM` 可使用自定义 Postgres 实例。`POSTGRES_URI_CUSTOM` 的值必须是有效的 [Postgres 连接 URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS)。

Postgres：

- 版本 15.8 或更高版本。
- 必须存在初始数据库，并且连接 URI 必须引用该数据库。

控制平面功能：

- 如果指定了 `POSTGRES_URI_CUSTOM`，LangGraph 控制平面将不会为服务器预配数据库。
- 如果删除了 `POSTGRES_URI_CUSTOM`，LangGraph 控制平面将不会为服务器预配数据库，也不会删除外部管理的 Postgres 实例。
- 如果删除了 `POSTGRES_URI_CUSTOM`，修订版的部署将不会成功。一旦指定了 `POSTGRES_URI_CUSTOM`，在其部署生命周期内必须始终设置该变量。
- 如果删除了部署，LangGraph 控制平面将不会删除外部管理的 Postgres 实例。
- `POSTGRES_URI_CUSTOM` 的值可以更新。例如，可以更新 URI 中的密码。

数据库连接：

- 自定义 Postgres 实例必须可被 LangGraph Server 访问。用户负责确保连接性。

## `REDIS_CLUSTER`

!!! info "仅允许用于自托管部署"
    Redis Cluster 模式仅适用于自托管部署模型，LangGraph Platform SaaS 将默认为您预配一个 redis 实例。

将 `REDIS_CLUSTER` 设置为 `True` 可启用 Redis Cluster 模式。启用后，系统将使用集群模式连接到 Redis。这在连接到 Redis Cluster 部署时非常有用。

默认为 `False`。

## `REDIS_KEY_PREFIX`

!!! info "API Server 版本 0.1.9+ 可用"
    API Server 版本 0.1.9 及更高版本支持此环境变量。

为 Redis 键指定一个前缀。这允许多个 LangGraph Server 实例通过使用不同的键前缀来共享同一个 Redis 实例。

默认为 `''`。

## `REDIS_URI_CUSTOM`

!!! info "仅适用于自托管数据平面和自托管控制平面"
    自定义 Redis 实例仅适用于 [自托管数据平面](../../concepts/langgraph_self_hosted_data_plane.md) 和 [自托管控制平面](../../concepts/langgraph_self_hosted_control_plane.md) 部署。

指定 `REDIS_URI_CUSTOM` 可使用自定义 Redis 实例。`REDIS_URI_CUSTOM` 的值必须是有效的 [Redis 连接 URI](https://redis-py.readthedocs.io/en/stable/connections.html#redis.Redis.from_url)。

## `RESUMABLE_STREAM_TTL_SECONDS`

Redis 中可恢复流数据的生存时间（以秒为单位）。

创建运行并流式传输输出时，可以将流配置为可恢复（例如 `stream_resumable=True`）。如果流可恢复，则流的输出将被临时存储在 Redis 中。此数据的 TTL 可以通过设置 `RESUMABLE_STREAM_TTL_SECONDS` 来配置。

有关如何实现可恢复流的更多详细信息，请参阅 [Python](https://langchain-ai.github.io/langgraph/cloud/reference/sdk/python_sdk_ref/#langgraph_sdk.client.RunsClient.stream) 和 [JS/TS](https://langchain-ai.github.io/langgraphjs/reference/classes/sdk_client.RunsClient.html#stream) SDK。

默认为 `120` 秒。