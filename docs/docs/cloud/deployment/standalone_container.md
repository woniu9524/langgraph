# 如何部署独立容器

部署前，请回顾独立容器的[概念指南](../../concepts/langgraph_standalone_container.md)部署选项。

## 先决条件

1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)在[本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)构建 Docker 镜像（例如 `langgraph build`）。
1. 独立容器部署需要以下环境变量。
    1. `REDIS_URI`: Redis 实例的连接详细信息。Redis 将用作发布/订阅代理，以实现后台运行的实时输出流式传输。`REDIS_URI` 的值必须是有效的[Redis 连接 URI](https://redis-py.readthedocs.io/en/stable/connections.html#redis.Redis.from_url)。

        !!! Note "共享 Redis 实例"
            多个自托管部署可以共享同一个 Redis 实例。例如，对于“部署 A”，可以将 `REDIS_URI` 设置为 `redis://<hostname_1>:<port>/1`，对于“部署 B”，可以将 `REDIS_URI` 设置为 `redis://<hostname_1>:<port>/2`。

            `1` 和 `2` 是同一实例中的不同数据库编号，但 `<hostname_1>` 是共享的。**不同的部署不能使用相同的数据库编号**。

    1. `DATABASE_URI`: Postgres 连接详细信息。Postgres 将用于存储助手、线程、运行、持久化线程状态和长期内存，以及管理具有“仅一次”语义的后台任务队列的状态。`DATABASE_URI` 的值必须是有效的[Postgres 连接 URI](https://www.postgresql.org/docs/current/libpq-connect.html#LIBPQ-CONNSTRING-URIS)。

        !!! Note "共享 Postgres 实例"
            多个自托管部署可以共享同一个 Postgres 实例。例如，对于“部署 A”，可以将 `DATABASE_URI` 设置为 `postgres://<user>:<password>@/<database_name_1>?host=<hostname_1>`，对于“部署 B”，可以将 `DATABASE_URI` 设置为 `postgres://<user>:<password>@/<database_name_2>?host=<hostname_1>`。

            `<database_name_1>` 和 `<database_name_2>` 是同一实例中的不同数据库，但 `<hostname_1>` 是共享的。**不同的部署不能使用相同的数据库**。

    1. `LANGSMITH_API_KEY`: （如果使用 [Lite](../../concepts/langgraph_server.md#server-versions)）LangSmith API 密钥。此密钥将在服务器启动时进行一次身份验证。
    1. `LANGGRAPH_CLOUD_LICENSE_KEY`: （如果使用 [Enterprise](../../concepts/langgraph_data_plane.md#licensing)）LangGraph Platform 许可证密钥。此密钥将在服务器启动时进行一次身份验证。
    1. `LANGSMITH_ENDPOINT`: 要将跟踪发送到[自托管 LangSmith](https://docs.smith.langchain.com/self_hosting) 实例，请将 `LANGSMITH_ENDPOINT` 设置为自托管 LangSmith 实例的主机名。

## Kubernetes (Helm)

使用此[Helm 图表](https://github.com/langchain-ai/helm/blob/main/charts/langgraph-cloud/README.md)将 LangGraph Server 部署到 Kubernetes 集群。

## Docker

运行以下 `docker` 命令：
```shell
docker run \
    --env-file .env \
    -p 8123:8000 \
    -e REDIS_URI="foo" \
    -e DATABASE_URI="bar" \
    -e LANGSMITH_API_KEY="baz" \
    my-image
```

!!! note

    * 您需要将 `my-image` 替换为您在先决条件步骤中构建的镜像名称（来自 `langgraph build`），并为 `REDIS_URI`、`DATABASE_URI` 和 `LANGSMITH_API_KEY` 提供适当的值。
    * 如果您的应用程序需要其他环境变量，可以通过类似方式传入。

## Docker Compose

Docker Compose YAML 文件：
```yml
volumes:
    langgraph-data:
        driver: local
services:
    langgraph-redis:
        image: redis:6
        healthcheck:
            test: redis-cli ping
            interval: 5s
            timeout: 1s
            retries: 5
    langgraph-postgres:
        image: postgres:16
        ports:
            - "5433:5432"
        environment:
            POSTGRES_DB: postgres
            POSTGRES_USER: postgres
            POSTGRES_PASSWORD: postgres
        volumes:
            - langgraph-data:/var/lib/postgresql/data
        healthcheck:
            test: pg_isready -U postgres
            start_period: 10s
            timeout: 1s
            retries: 5
            interval: 5s
    langgraph-api:
        image: ${IMAGE_NAME}
        ports:
            - "8123:8000"
        depends_on:
            langgraph-redis:
                condition: service_healthy
            langgraph-postgres:
                condition: service_healthy
        env_file:
            - .env
        environment:
            REDIS_URI: redis://langgraph-redis:6379
            LANGSMITH_API_KEY: ${LANGSMITH_API_KEY}
            POSTGRES_URI: postgres://postgres:postgres@langgraph-postgres:5432/postgres?sslmode=disable
```

您可以在同一个文件夹中运行带有此 Docker Compose 文件的命令 `docker compose up`。

这将启动一个端口为 `8123` 的 LangGraph Server（如果您想更改此端口，可以通过更改 `langgraph-api` 卷中的端口来更改）。您可以通过运行以下命令来测试应用程序是否正常运行：

```shell
curl --request GET --url 0.0.0.0:8123/ok
```
如果一切运行正常，您应该会看到类似以下的回应：

```shell
{"ok":true}
```