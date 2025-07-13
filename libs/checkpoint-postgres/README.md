# LangGraph Checkpoint Postgres

LangGraph CheckpointSaver 的实现，使用 Postgres。

## 依赖

默认情况下，`langgraph-checkpoint-postgres` 会安装不带任何附加组件的 `psycopg`（Psycopg 3）。但是，您可以选择最适合您需求的特定安装版本，有关说明请参见 [此处](https://www.psycopg.org/psycopg3/docs/basic/install.html)（例如 `psycopg[binary]`）。

## 用法

> [!IMPORTANT]
> 首次使用 Postgres checkpointer 时，请务必调用其 `.setup()` 方法来创建所需的表。请参见下面的示例。

> [!IMPORTANT]
> 在手动创建 Postgres 连接并将其传递给 `PostgresSaver` 或 `AsyncPostgresSaver` 时，请确保包含 `autocommit=True` 和 `row_factory=dict_row`（`from psycopg.rows import dict_row`）。完整的示例请参见此 [操作指南](https://langchain-ai.github.io/langgraph/how-tos/persistence_postgres/)。
>
> **为什么需要这些参数：**
> - `autocommit=True`：`.setup()` 方法需要此参数才能将检查点表正确提交到数据库。否则，表创建可能不会被持久化。
> - `row_factory=dict_row`：PostgresSaver 实现使用类似字典的语法（例如 `row["column_name"]`）访问数据库行，因此需要此参数。默认的 `tuple_row` 工厂返回仅支持基于索引的访问（例如 `row[0]`）的元组，当 checkpointer 尝试按名称访问列时，这将导致 `TypeError` 异常。
>
> **错误用法示例：**
> ```python
> # ❌ 这将在 checkpointer 操作期间因 TypeError 而失败
> with psycopg.connect(DB_URI) as conn:  # 缺少 autocommit=True 和 row_factory=dict_row
>     checkpointer = PostgresSaver(conn)
>     checkpointer.setup()  # 表可能无法正确持久化
>     # 任何从数据库读取的操作都会失败并出现以下错误：
>     # TypeError: tuple indices must be integers or slices, not str
> ```

```python
from langgraph.checkpoint.postgres import PostgresSaver

write_config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
read_config = {"configurable": {"thread_id": "1"}}

DB_URI = "postgres://postgres:postgres@localhost:5432/postgres?sslmode=disable"
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    # 首次使用 checkpointer 时调用 .setup()
    checkpointer.setup()
    checkpoint = {
        "v": 4,
        "ts": "2024-07-31T20:14:19.804150+00:00",
        "id": "1ef4f797-8335-6428-8001-8a1503f9b875",
        "channel_values": {
            "my_key": "meow",
            "node": "node"
        },
        "channel_versions": {
            "__start__": 2,
            "my_key": 3,
            "start:node": 3,
            "node": 3
        },
        "versions_seen": {
            "__input__": {},
            "__start__": {
            "__start__": 1
            },
            "node": {
            "start:node": 2
            }
        },
    }

    # 存储 checkpoint
    checkpointer.put(write_config, checkpoint, {}, {})

    # 加载 checkpoint
    checkpointer.get(read_config)

    # 列出 checkpoints
    list(checkpointer.list(read_config))
```

### Async

```python
from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:
    checkpoint = {
        "v": 4,
        "ts": "2024-07-31T20:14:19.804150+00:00",
        "id": "1ef4f797-8335-6428-8001-8a1503f9b875",
        "channel_values": {
            "my_key": "meow",
            "node": "node"
        },
        "channel_versions": {
            "__start__": 2,
            "my_key": 3,
            "start:node": 3,
            "node": 3
        },
        "versions_seen": {
            "__input__": {},
            "__start__": {
            "__start__": 1
            },
            "node": {
            "start:node": 2
            }
        },
    }

    # 存储 checkpoint
    await checkpointer.aput(write_config, checkpoint, {}, {})

    # 加载 checkpoint
    await checkpointer.aget(read_config)

    # 列出 checkpoints
    [c async for c in checkpointer.alist(read_config)]
```