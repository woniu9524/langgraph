# LangGraph Checkpoint

本库定义了 LangGraph checkpointer 的基础接口。Checkpointer 提供了一个持久化层，用于 LangGraph。它们允许您与图的状态进行交互和管理。当您使用带有 checkpointer 的图时，checkpointer 会在每个 superstep 保存图状态的 _checkpoint_，从而实现诸如 人工干预（human-in-the-loop）、交互之间的"记忆"等强大的功能。

## 核心概念

### Checkpoint

Checkpoint 是图在给定时间点的状态快照。Checkpoint tuple 指的是一个包含 checkpoint 及其关联配置、元数据和待处理写入的对象。

### Thread

Threads 能够对多个不同的运行进行 checkpoint，这对于多租户聊天应用程序和其他需要维护独立状态的场景至关重要。Thread 是分配给 checkpointer 保存的一系列 checkpoint 的唯一 ID。当使用 checkpointer 时，您必须在运行图时指定 `thread_id`，并可选择性地指定 `checkpoint_id`。

- `thread_id` 只是一个 thread 的 ID。此项始终必需。
- `checkpoint_id` 可选。此标识符引用 thread 中的特定 checkpoint。可用于从 thread 中某个中间点开始运行图。

在调用图时，您必须将这些作为配置的可配置部分传递，例如：

```python
{"configurable": {"thread_id": "1"}}  # 有效配置
{"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}  # 也有效配置
```

### Serde

`langgraph_checkpoint` 还定义了序列化/反序列化 (serde) 协议，并提供了一个默认实现（`langgraph.checkpoint.serde.jsonplus.JsonPlusSerializer`），该实现能够处理包括 LangChain 和 LangGraph 原始类型、datetime、enums 等在内的多种多样的类型。

### Pending writes

当图节点在给定 superstep 的执行过程中失败时，LangGraph 会存储来自该 superstep 中其他成功完成的节点的待处理 checkpoint 写入，以便每当我们从该 superstep 恢复图执行时，都不会重新运行成功的节点。

## Interface

每个 checkpointer 都应符合 `langgraph.checkpoint.base.BaseCheckpointSaver` 接口，并且必须实现以下方法：

- `.put` - 存储一个带有其配置和元数据的 checkpoint。
- `.put_writes` - 存储与 checkpoint 相关联的中间写入（即 pending 写入）。
- `.get_tuple` - 使用给定的配置（`thread_id` 和 `checkpoint_id`）获取一个 checkpoint tuple。
- `.list` - 列出与给定配置和过滤条件匹配的 checkpoints。

如果 checkpointer 将与异步图执行一起使用（即通过 `.ainvoke`、`.astream`、`.abatch` 执行图），checkpointer 必须实现上述方法的异步版本（`.aput`、`.aput_writes`、`.aget_tuple`、`.alist`）。

## Usage

```python
from langgraph.checkpoint.memory import InMemorySaver

write_config = {"configurable": {"thread_id": "1", "checkpoint_ns": ""}}
read_config = {"configurable": {"thread_id": "1"}}

checkpointer = InMemorySaver()
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