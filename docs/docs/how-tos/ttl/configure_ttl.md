# 为 LangGraph 应用添加 TTL

!!! tip "先决条件"

    本指南假设您已熟悉 [LangGraph 平台](../../concepts/langgraph_platform.md)、[持久化](../../concepts/persistence.md) 和 [跨线程持久化](../../concepts/persistence.md#memory-store) 的概念。

???+ note "仅限 LangGraph 平台"
    
    TTL（Time-to-Live，生存时间）仅支持 LangGraph 平台部署。本指南不适用于 LangGraph OSS。

LangGraph 平台会持久化[检查点](../../concepts/persistence.md#checkpoints)（线程状态）和[跨线程内存](../../concepts/persistence.md#memory-store)（存储项）。在 `langgraph.json` 中配置生存时间（TTL）策略，可以自动管理这些数据的生命周期，防止数据无限累积。

## 配置检查点 TTL

检查点会捕获对话线程的状态。设置 TTL 可以确保旧的检查点和线程被自动删除。

在 `langgraph.json` 文件中添加 `checkpointer.ttl` 配置：

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  }
}
```
:::

*   `strategy`: 指定过期后采取的操作。目前仅支持 `"delete"`，该选项会在检查点过期时删除该线程下的所有检查点。
*   `sweep_interval_minutes`: 定义系统检查过期检查点的频率（分钟）。
*   `default_ttl`: 设置检查点的默认生存时间（分钟），例如 43200 分钟 = 30 天。

## 配置存储项 TTL

存储项允许跨线程进行数据持久化。配置存储项的 TTL 有助于通过删除过时数据来管理内存。

在 `langgraph.json` 文件中添加 `store.ttl` 配置：

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

:::js
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

*   `refresh_on_read`: （可选，默认为 `true`）如果设置为 `true`，通过 `get` 或 `search` 访问项会重置其过期计时器。如果设置为 `false`，TTL 仅在 `put` 时刷新。
*   `sweep_interval_minutes`: （可选）定义系统检查过期项的频率（分钟）。如果省略，则不会进行扫描。
*   `default_ttl`: （可选）设置存储项的默认生存时间（分钟），例如 10080 分钟 = 7 天。如果省略，则项默认不会过期。

## 组合 TTL 配置

您可以在同一个 `langgraph.json` 文件中配置检查点和存储项的 TTL，为不同类型的数据设置不同的策略。这是一个示例：

:::python
```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.py:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

:::js
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./agent.ts:graph"
  },
  "checkpointer": {
    "ttl": {
      "strategy": "delete",
      "sweep_interval_minutes": 60,
      "default_ttl": 43200
    }
  },
  "store": {
    "ttl": {
      "refresh_on_read": true,
      "sweep_interval_minutes": 120,
      "default_ttl": 10080
    }
  }
}
```
:::

## 运行时覆盖

`langgraph.json` 中的默认 `store.ttl` 设置可以通过在 SDK 方法调用（如 `get`、`put` 和 `search`）中提供特定的 TTL 值来在运行时覆盖。

## 部署过程

在 `langgraph.json` 中配置好 TTL 后，请部署或重启您的 LangGraph 应用程序以使更改生效。使用 `langgraph dev` 进行本地开发，或使用 `langgraph up` 进行 Docker 部署。

有关其他可配置选项的更多详细信息，请参阅 @[langgraph.json CLI 参考][langgraph.json]。