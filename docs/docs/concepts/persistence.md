---
search:
  boost: 2
---

# 持久化

LangGraph 内置了一个持久化层，通过 checkpointers（检查点）实现。当你使用 checkpointer 编译图时，checkpointer 会在每个超级步（super-step）保存图状态的 `checkpoint`（检查点）。这些检查点会被保存到一个 `thread`（线程）中，可以在图执行后访问。由于 `threads` 允许在执行后访问图的状态，因此可以实现诸如人工干预（human-in-the-loop）、记忆（memory）、时间旅行（time travel）和容错（fault-tolerance）等强大的功能。下面我们将详细讨论这些概念。

![Checkpoints](img/persistence/checkpoints.jpg)

!!! info "LangGraph API 自动处理检查点"

    使用 LangGraph API 时，您无需手动实现或配置 checkpointers。API 会在后台为您处理所有持久化基础设施。

## 线程 (Threads)

线程是 checkpointer 保存的每个检查点分配到的唯一 ID 或线程标识符。它包含一系列 [运行](./assistants.md#execution) 的累积状态。当一个运行被执行时，助手的底层图的状态将被持久化到该线程。

当调用带有 checkpointer 的图时，您**必须** 在 `configurable` 配置部分指定 `thread_id`：

:::python

```python
{"configurable": {"thread_id": "1"}}
```

:::

:::js

```typescript
{
  configurable: {
    thread_id: "1";
  }
}
```

:::

可以检索线程的当前状态和历史状态。要持久化状态，必须在执行运行之前创建线程。LangGraph Platform API 提供了多个用于创建和管理线程及线程状态的端点。有关更多详细信息，请参阅 [API 参考](../cloud/reference/api/api_ref.html#tag/threads)。

## 检查点 (Checkpoints)

线程在特定时间点的状态称为检查点。检查点是在每个超级步保存的图状态的快照，由 `StateSnapshot` 对象表示，该对象具有以下关键属性：

- `config`: 此检查点关联的配置。
- `metadata`: 此检查点关联的元数据。
- `values`: 此时间点的状态通道值。
- `next`: 要在图中执行的节点名称的元组。
- `tasks`: 包含有关要执行的下一个任务信息的 `PregelTask` 对象元组。如果该步骤之前已被尝试过，它将包含错误信息。如果图在节点内部被[动态](../how-tos/human_in_the_loop/add-human-in-the-loop.md#pause-using-interrupt)中断，`tasks` 将包含与中断相关的附加数据。

检查点会被持久化，并可用于稍后恢复线程的状态。

让我们看看调用简单图时会保存哪些检查点：

:::python

```python
from langgraph.graph import StateGraph, START, END
from langgraph.checkpoint.memory import InMemorySaver
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: str
    bar: Annotated[list[str], add]

def node_a(state: State):
    return {"foo": "a", "bar": ["a"]}

def node_b(state: State):
    return {"foo": "b", "bar": ["b"]}


workflow = StateGraph(State)
workflow.add_node(node_a)
workflow.add_node(node_b)
workflow.add_edge(START, "node_a")
workflow.add_edge("node_a", "node_b")
workflow.add_edge("node_b", END)

checkpointer = InMemorySaver()
graph = workflow.compile(checkpointer=checkpointer)

config = {"configurable": {"thread_id": "1"}}
graph.invoke({"foo": ""}, config)
```

:::

:::js

```typescript
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

:::

:::js

```typescript
import { StateGraph, START, END, MemoryServer } from "@langchain/langgraph";
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.string(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});

const workflow = new StateGraph(State)
  .addNode("nodeA", (state) => {
    return { foo: "a", bar: ["a"] };
  })
  .addNode("nodeB", (state) => {
    return { foo: "b", bar: ["b"] };
  })
  .addEdge(START, "nodeA")
  .addEdge("nodeA", "nodeB")
  .addEdge("nodeB", END);

const checkpointer = new MemorySaver();
const graph = workflow.compile({ checkpointer });

const config = { configurable: { thread_id: "1" } };
await graph.invoke({ foo: "" }, config);
```

:::

:::python

运行图后，预计会看到 4 个检查点：

- 带有 `START` 作为下一个执行节点的空检查点
- 带有用户输入 `{'foo': '', 'bar': []}` 以及 `node_a` 作为下一个执行节点的检查点
- 带有 `node_a` 输出 `{'foo': 'a', 'bar': ['a']}` 以及 `node_b` 作为下一个执行节点的检查点
- 带有 `node_b` 输出 `{'foo': 'b', 'bar': ['a', 'b']}` 以及无下一个执行节点的检查点

请注意，由于我们为 `bar` 通道设置了 reducer，因此 `bar` 通道的值包含了来自两个节点的输出。

:::

:::js

运行图后，预计会看到 4 个检查点：

- 带有 `START` 作为下一个执行节点的空检查点
- 带有用户输入 `{'foo': '', 'bar': []}` 以及 `nodeA` 作为下一个执行节点的检查点
- 带有 `nodeA` 输出 `{'foo': 'a', 'bar': ['a']}` 以及 `nodeB` 作为下一个执行节点的检查点
- 带有 `nodeB` 输出 `{'foo': 'b', 'bar': ['a', 'b']}` 以及无下一个执行节点的检查点

请注意，由于我们为 `bar` 通道设置了 reducer，因此 `bar` 通道的值包含了来自两个节点的输出。
:::

### 获取状态

:::python
在与保存的图状态交互时，您**必须** 指定一个[线程标识符](#threads)。您可以通过调用 `graph.get_state(config)` 来查看图的_最新_状态。这将返回一个 `StateSnapshot` 对象，该对象对应于配置中提供的线程 ID 的最新检查点，或者对应于线程的特定检查点 ID 的检查点（如果已提供）。

```python
# 获取最新状态快照
config = {"configurable": {"thread_id": "1"}}
graph.get_state(config)

# 获取特定 checkpoint_id 的状态快照
config = {"configurable": {"thread_id": "1", "checkpoint_id": "1ef663ba-28fe-6528-8002-5a559208592c"}}
graph.get_state(config)
```

:::

:::js
在与保存的图状态交互时，您**必须** 指定一个[线程标识符](#threads)。您可以通过调用 `graph.getState(config)` 来查看图的_最新_状态。这将返回一个 `StateSnapshot` 对象，该对象对应于配置中提供的线程 ID 的最新检查点，或者对应于线程的特定检查点 ID 的检查点（如果已提供）。

```typescript
// 获取最新状态快照
const config = { configurable: { thread_id: "1" } };
await graph.getState(config);

// 获取特定 checkpoint_id 的状态快照
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "1ef663ba-28fe-6528-8002-5a559208592c",
  },
};
await graph.getState(config);
```

:::

:::python

在我们的示例中，`get_state` 的输出将如下所示：

```
StateSnapshot(
    values={'foo': 'b', 'bar': ['a', 'b']},
    next=(),
    config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
    metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
    created_at='2024-08-29T19:19:38.821749+00:00',
    parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}}, tasks=()
)
```

:::

:::js

在我们的示例中，`getState` 的输出将如下所示：

```
StateSnapshot {
  values: { foo: 'b', bar: ['a', 'b'] },
  next: [],
  config: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
    }
  },
  metadata: {
    source: 'loop',
    writes: { nodeB: { foo: 'b', bar: ['b'] } },
    step: 2
  },
  createdAt: '2024-08-29T19:19:38.821749+00:00',
  parentConfig: {
    configurable: {
      thread_id: '1',
      checkpoint_ns: '',
      checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
    }
  },
  tasks: []
}
```

:::

### 获取状态历史

:::python
您可以通过调用 `graph.get_state_history(config)` 来获取指定线程的完整图执行历史。这将返回一个与配置中提供的线程 ID 关联的 `StateSnapshot` 对象列表。重要的是，检查点将按时间顺序排列，最新的检查点/`StateSnapshot` 对象将位于列表的第一个位置。

```python
config = {"configurable": {"thread_id": "1"}}
list(graph.get_state_history(config))
```

:::

:::js
您可以通过调用 `graph.getStateHistory(config)` 来获取指定线程的完整图执行历史。这将返回一个与配置中提供的线程 ID 关联的 `StateSnapshot` 对象列表。重要的是，检查点将按时间顺序排列，最新的检查点/`StateSnapshot` 对象将位于列表的第一个位置。

```typescript
const config = { configurable: { thread_id: "1" } };
for await (const state of graph.getStateHistory(config)) {
  console.log(state);
}
```

:::

:::python

在我们的示例中，`get_state_history` 的输出将如下所示：

```
[
    StateSnapshot(
        values={'foo': 'b', 'bar': ['a', 'b']},
        next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28fe-6528-8002-5a559208592c'}},
        metadata={'source': 'loop', 'writes': {'node_b': {'foo': 'b', 'bar': ['b']}}, 'step': 2},
        created_at='2024-08-29T19:19:38.821749+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        tasks=(),
    ),
    StateSnapshot(
        values={'foo': 'a', 'bar': ['a']},
        next=('node_b',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f9-6ec4-8001-31981c2c39f8'}},
        metadata={'source': 'loop', 'writes': {'node_a': {'foo': 'a', 'bar': ['a']}}, 'step': 1},
        created_at='2024-08-29T19:19:38.819946+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        tasks=(PregelTask(id='6fb7314f-f114-5413-a1f3-d37dfe98ff44', name='node_b', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'foo': '', 'bar': []},
        next=('node_a',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f4-6b4a-8000-ca575a13d36a'}},
        metadata={'source': 'loop', 'writes': None, 'step': 0},
        created_at='2024-08-29T19:19:38.817813+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        tasks=(PregelTask(id='f1b14528-5ee5-579c-949b-23ef9bfbed58', name='node_a', error=None, interrupts=()),),
    ),
    StateSnapshot(
        values={'bar': []},
        next=('__start__',),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1ef663ba-28f0-6c66-bfff-6723431e8481'}},
        metadata={'source': 'input', 'writes': {'foo': ''}, 'step': -1},
        created_at='2024-08-29T19:19:38.816205+00:00',
        parent_config=None,
        tasks=(PregelTask(id='6d27aa2e-d72b-5504-a36f-8620e54a76dd', name='__start__', error=None, interrupts=()),),
    )
]
```

:::

:::js

在我们的示例中，`getStateHistory` 的输出将如下所示：

```
[
  StateSnapshot {
    values: { foo: 'b', bar: ['a', 'b'] },
    next: [],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28fe-6528-8002-5a559208592c'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeB: { foo: 'b', bar: ['b'] } },
      step: 2
    },
    createdAt: '2024-08-29T19:19:38.821749+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    tasks: []
  },
  StateSnapshot {
    values: { foo: 'a', bar: ['a'] },
    next: ['nodeB'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f9-6ec4-8001-31981c2c39f8'
      }
    },
    metadata: {
      source: 'loop',
      writes: { nodeA: { foo: 'a', bar: ['a'] } },
      step: 1
    },
    createdAt: '2024-08-29T19:19:38.819946+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    tasks: [
      PregelTask {
        id: '6fb7314f-f114-5413-a1f3-d37dfe98ff44',
        name: 'nodeB',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { foo: '', bar: [] },
    next: ['node_a'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f4-6b4a-8000-ca575a13d36a'
      }
    },
    metadata: {
      source: 'loop',
      writes: null,
      step: 0
    },
    createdAt: '2024-08-29T19:19:38.817813+00:00',
    parentConfig: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    tasks: [
      PregelTask {
        id: 'f1b14528-5ee5-579c-949b-23ef9bfbed58',
        name: 'node_a',
        error: null,
        interrupts: []
      }
    ]
  },
  StateSnapshot {
    values: { bar: [] },
    next: ['__start__'],
    config: {
      configurable: {
        thread_id: '1',
        checkpoint_ns: '',
        checkpoint_id: '1ef663ba-28f0-6c66-bfff-6723431e8481'
      }
    },
    metadata: {
      source: 'input',
      writes: { foo: '' },
      step: -1
    },
    createdAt: '2024-08-29T19:19:38.816205+00:00',
    parentConfig: null,
    tasks: [
      PregelTask {
        id: '6d27aa2e-d72b-5504-a36f-8620e54a76dd',
        name: '__start__',
        error: null,
        interrupts: []
      }
    ]
  }
]
```

:::

![State](img/persistence/get_state.jpg)

### 回放 (Replay)

也可以回放之前的图执行。如果我们使用 `thread_id` 和 `checkpoint_id` 来 `invoke` 图，那么我们将在对应于 `checkpoint_id` 的检查点之前的步骤被_回放_，并且只执行检查点之后的步骤。

- `thread_id` 是线程的 ID。
- `checkpoint_id` 是指向线程中特定检查点的标识符。

在调用图时，您必须在配置的 `configurable` 部分传递这些参数：

:::python

```python
config = {"configurable": {"thread_id": "1", "checkpoint_id": "0c62ca34-ac19-445d-bbb0-5b4984975b2a"}}
graph.invoke(None, config=config)
```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
    checkpoint_id: "0c62ca34-ac19-445d-bbb0-5b4984975b2a",
  },
};
await graph.invoke(null, config);
```

:::

重要的是，LangGraph 知道某个步骤是否之前已被执行过。如果已经执行过，LangGraph 将_回放_该步骤，而不会重新执行它，仅限于_提供 `checkpoint_id` 之前的_步骤。`checkpoint_id` 之后的_所有步骤都将被执行（即，创建新的分支），即使它们之前已被执行过。请参阅此[操作指南，了解时间旅行以及如何进行回放](../how-tos/human_in_the_loop/time-travel.md)。

![Replay](img/persistence/re_play.png)

### 更新状态

:::python

除了从特定的 `checkpoints` 回放图之外，我们还可以_编辑_图的状态。我们通过使用 `graph.update_state()` 来实现。此方法接受三个不同的参数：

:::

:::js

除了从特定的 `checkpoints` 回放图之外，我们还可以_编辑_图的状态。我们通过使用 `graph.updateState()` 来实现。此方法接受三个不同的参数：

:::

#### `config`

配置应包含 `thread_id`，用于指定要更新的线程。当仅传递 `thread_id` 时，我们会更新（或分支）当前状态。如果选择包含 `checkpoint_id` 字段，则会分支该选定的检查点。

#### `values`

这些是用于更新状态的值。请注意，此更新的处理方式与来自节点的任何更新都完全相同。这意味着这些值将被传递给[ reducer](./low_level.md#reducers) 函数（如果它们为图状态中的某些通道定义了）。这意味着 `update_state` **不会** 自动覆盖每个通道的通道值，而只会更新没有 reducer 的通道。让我们通过一个示例来理解。

假设您已使用以下架构定义了图的状态（请参阅上面的完整示例）：

:::python

```python
from typing import Annotated
from typing_extensions import TypedDict
from operator import add

class State(TypedDict):
    foo: int
    bar: Annotated[list[str], add]
```

:::

:::js

```typescript
import { withLangGraph } from "@langchain/langgraph/zod";
import { z } from "zod";

const State = z.object({
  foo: z.number(),
  bar: withLangGraph(z.array(z.string()), {
    reducer: {
      fn: (x, y) => x.concat(y),
    },
    default: () => [],
  }),
});
```

:::

现在，假设图的当前状态为：

:::python

```
{"foo": 1, "bar": ["a"]}
```

:::

:::js

```typescript
{ foo: 1, bar: ["a"] }
```

:::

如果您按如下方式更新状态：

:::python

```python
graph.update_state(config, {"foo": 2, "bar": ["b"]})
```

:::

:::js

```typescript
await graph.updateState(config, { foo: 2, bar: ["b"] });
```

:::

那么图的新状态将是：

:::python

```
{"foo": 2, "bar": ["a", "b"]}
```

`foo` 键（通道）被完全更改（因为该通道未指定 reducer，所以 `update_state` 会覆盖它）。但是，为 `bar` 键指定了 reducer，因此它会将 `"b"` 追加到 `bar` 的状态中。
:::

:::js

```typescript
{ foo: 2, bar: ["a", "b"] }
```

`foo` 键（通道）被完全更改（因为该通道未指定 reducer，所以 `updateState` 会覆盖它）。但是，为 `bar` 键指定了 reducer，因此它会将 `"b"` 追加到 `bar` 的状态中。
:::

#### `as_node`

:::python
在调用 `update_state` 时，您可以选择指定的最后一个参数是 `as_node`。如果您提供了它，则更新将作为来自 `as_node` 节点的更新应用。如果未提供 `as_node`，则它将设置为更新状态的最后一个节点（如果不存在歧义）。之所以重要，是因为下一个要执行的步骤取决于最后一个更新状态的节点，因此这可用于控制哪个节点接下来执行。请参阅此[时间旅行操作指南，了解更多关于分支状态的信息](../how-tos/human_in_the_loop/time-travel.md)。
:::

:::js
在调用 `updateState` 时，您可以选择指定的最后一个参数是 `asNode`。如果您提供了它，则更新将作为来自 `asNode` 节点的更新应用。如果未提供 `asNode`，则它将设置为更新状态的最后一个节点（如果不存在歧义）。之所以重要，是因为下一个要执行的步骤取决于最后一个更新状态的节点，因此这可用于控制哪个节点接下来执行。请参阅此[时间旅行操作指南，了解更多关于分支状态的信息](../how-tos/human_in_the_loop/time-travel.md)。
:::

![Update](img/persistence/checkpoints_full_story.jpg)

## 内存存储 (Memory Store)

![共享状态模型](img/persistence/shared_state.png)

[状态模式](low_level.md#schema) 指定了一组在图执行过程中填充的键。如上所述，状态可以由 checkpointer 在每个图步骤中写入线程，从而实现状态持久化。

但是，如果我们想保留一些信息_跨线程_共享呢？考虑一个聊天机器人，我们希望保留该用户在所有（线程）聊天对话中的特定信息（例如，用户名）！

仅使用 checkpointers，我们无法在跨线程共享信息。这就引出了 [`Store`](../reference/store.md#langgraph.store.base.BaseStore) 接口的必要性。作为说明，我们可以定义一个 `InMemoryStore` 来存储有关用户的信息，这些信息可以跨线程共享。我们像之前一样，编译图并使用 checkpointer 和我们新创建的 `in_memory_store` 变量。

!!! info "LangGraph API 自动处理存储"

    使用 LangGraph API 时，您无需手动实现或配置存储。API 会为您处理所有存储基础设施。

### 基本用法

首先，让我们在不使用 LangGraph 的情况下单独展示这一点。

:::python

```python
from langgraph.store.memory import InMemoryStore
in_memory_store = InMemoryStore()
```

:::

:::js

```typescript
import { MemoryStore } from "@langchain/langgraph";

const memoryStore = new MemoryStore();
```

:::

记忆（Memories）按`tuple`进行命名空间划分，在这个特定示例中是 `(<user_id>, "memories")`。命名空间可以是任意长度，代表任何事物，不一定是用户特定的。

:::python

```python
user_id = "1"
namespace_for_memory = (user_id, "memories")
```

:::

:::js

```typescript
const userId = "1";
const namespaceForMemory = [userId, "memories"];
```

:::

我们使用 `store.put` 方法将记忆保存到命名空间中。执行此操作时，我们指定命名空间（如上定义）以及记忆的键值对：键是记忆的唯一标识符（`memory_id`），值（一个字典）是记忆本身。

:::python

```python
memory_id = str(uuid.uuid4())
memory = {"food_preference" : "I like pizza"}
in_memory_store.put(namespace_for_memory, memory_id, memory)
```

:::

:::js

```typescript
import { v4 as uuidv4 } from "uuid";

const memoryId = uuidv4();
const memory = { food_preference: "I like pizza" };
await memoryStore.put(namespaceForMemory, memoryId, memory);
```

:::

我们可以使用 `store.search` 方法从命名空间中读取记忆，该方法将返回该用户的所有记忆，形式为一个列表。最新的记忆位于列表的末尾。

:::python

```python
memories = in_memory_store.search(namespace_for_memory)
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

每个记忆类型都是一个 Python 类（[`Item`](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.Item)），具有某些属性。我们可以通过如上所示的 `.dict()` 进行转换，以字典形式访问它。

它具有的属性是：

- `value`: 此记忆的值（本身是一个字典）。
- `key`: 此命名空间中此记忆的唯一键。
- `namespace`: 字符串列表，此记忆类型的命名空间。
- `created_at`: 此记忆创建时间戳。
- `updated_at`: 此记忆更新时间戳。

:::

:::js

```typescript
const memories = await memoryStore.search(namespaceForMemory);
memories[memories.length - 1];

// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

它具有的属性是：

- `value`: 此记忆的值。
- `key`: 此命名空间中此记忆的唯一键。
- `namespace`: 字符串列表，此记忆类型的命名空间。
- `createdAt`: 此记忆创建时间戳。
- `updatedAt`: 此记忆更新时间戳。

:::

### 语义搜索

除了简单的检索之外，存储还支持语义搜索，允许您根据含义而不是精确匹配来查找记忆。要启用此功能，请使用嵌入模型配置存储：

:::python

```python
from langchain.embeddings import init_embeddings

store = InMemoryStore(
    index={
        "embed": init_embeddings("openai:text-embedding-3-small"),  # Embedding provider
        "dims": 1536,                              # Embedding dimensions
        "fields": ["food_preference", "$"]              # Fields to embed
    }
)
```

:::

:::js

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";

const store = new InMemoryStore({
  index: {
    embeddings: new OpenAIEmbeddings({ model: "text-embedding-3-small" }),
    dims: 1536,
    fields: ["food_preference", "$"], // Fields to embed
  },
});
```

:::

现在搜索时，您可以使用自然语言查询来查找相关的记忆：

:::python

```python
# 查找关于食物偏好的记忆
# （这可以在将记忆放入存储后进行）
memories = store.search(
    namespace_for_memory,
    query="What does the user like to eat?",
    limit=3  # 返回前 3 个匹配项
)
```

:::

:::js

```typescript
// 查找关于食物偏好的记忆
// （这可以在将记忆放入存储后进行）
const memories = await store.search(namespaceForMemory, {
  query: "What does the user like to eat?",
  limit: 3, // 返回前 3 个匹配项
});
```

:::

您可以通过配置 `fields` 参数或在存储记忆时指定 `index` 参数来控制要嵌入的记忆部分：

:::python

```python
# 使用特定字段进行存储
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {
        "food_preference": "I love Italian cuisine",
        "context": "Discussing dinner plans"
    },
    index=["food_preference"]  # 仅嵌入 "food_preferences" 字段
)

# 无嵌入存储（仍可检索，但不可搜索）
store.put(
    namespace_for_memory,
    str(uuid.uuid4()),
    {"system_info": "Last updated: 2024-01-01"},
    index=False
)
```

:::

:::js

```typescript
// 使用特定字段进行存储
await store.put(
  namespaceForMemory,
  uuidv4(),
  {
    food_preference: "I love Italian cuisine",
    context: "Discussing dinner plans",
  },
  { index: ["food_preference"] } // 仅嵌入 "food_preferences" 字段
);

// 无嵌入存储（仍可检索，但不可搜索）
await store.put(
  namespaceForMemory,
  uuidv4(),
  { system_info: "Last updated: 2024-01-01" },
  { index: false }
);
```

:::

### 在 LangGraph 中使用

:::python
一切就绪后，我们在 LangGraph 中使用 `in_memory_store`。`in_memory_store` 与 checkpointer 协同工作：checkpointer 将状态保存到线程，如上所述，而 `in_memory_store` 允许我们存储任意信息以供_跨线程_访问。我们像这样编译图，同时使用 checkpointer 和 `in_memory_store`。

```python
from langgraph.checkpoint.memory import InMemorySaver

# 我们需要这个，因为我们要启用线程（对话）
checkpointer = InMemorySaver()

# ... 定义图 ...

# 使用 checkpointer 和 store 编译图
graph = graph.compile(checkpointer=checkpointer, store=in_memory_store)
```

:::

:::js

一切就绪后，我们在 LangGraph 中使用 `memoryStore`。`memoryStore` 与 checkpointer 协同工作：checkpointer 将状态保存到线程，如上所述，而 `memoryStore` 允许我们存储任意信息以供_跨线程_访问。我们像这样编译图，同时使用 checkpointer 和 `memoryStore`。

```typescript
import { MemorySaver } from "@langchain/langgraph";

// 我们需要这个，因为我们要启用线程（对话）
const checkpointer = new MemorySaver();

// ... 定义图 ...

// 使用 checkpointer 和 store 编译图
const graph = workflow.compile({ checkpointer, store: memoryStore });
```

:::

我们使用 `thread_id` 调用图，就像之前一样，同时还使用 `user_id`，我们将使用它来为我们的记忆命名空间，正如我们上面所示。

:::python

```python
# 调用图
user_id = "1"
config = {"configurable": {"thread_id": "1", "user_id": user_id}}

# 首先，我们向 AI 打个招呼
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi"}]}, config, stream_mode="updates"
):
    print(update)
```

:::

:::js

```typescript
// 调用图
const userId = "1";
const config = { configurable: { thread_id: "1", user_id: userId } };

// 首先，我们向 AI 打个招呼
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

:::

:::python
我们可以在_任何节点_中访问 `in_memory_store` 和 `user_id`，方法是传递 `store: BaseStore` 和 `config: RunnableConfig` 作为节点参数。以下是我们如何在节点中使用语义搜索来查找相关记忆的方法：

```python
def update_memory(state: MessagesState, config: RunnableConfig, *, store: BaseStore):

    # 从配置中获取用户 ID
    user_id = config["configurable"]["user_id"]

    # 对记忆进行命名空间划分
    namespace = (user_id, "memories")

    # ... 分析对话并创建新记忆

    # 创建新的记忆 ID
    memory_id = str(uuid.uuid4())

    # 我们创建一个新记忆
    store.put(namespace, memory_id, {"memory": memory})

```

:::

:::js
我们可以在_任何节点_中访问 `memoryStore` 和 `user_id`，方法是访问 `config` 和 `store` 作为节点参数。以下是我们如何在节点中使用语义搜索来查找相关记忆的方法：

```typescript
import {
  LangGraphRunnableConfig,
  BaseStore,
  MessagesZodState,
} from "@langchain/langgraph";
import { z } from "zod";

const updateMemory = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // 从配置中获取用户 ID
  const userId = config.configurable?.user_id;

  // 对记忆进行命名空间划分
  const namespace = [userId, "memories"];

  // ... 分析对话并创建新记忆

  // 创建新的记忆 ID
  const memoryId = uuidv4();

  // 我们创建一个新记忆
  await store.put(namespace, memoryId, { memory });
};
```

:::

如上所示，我们还可以访问任意节点中的存储，并使用 `store.search` 方法来获取记忆。回想一下，记忆以对象列表的形式返回，可以转换为字典。

:::python

```python
memories[-1].dict()
{'value': {'food_preference': 'I like pizza'},
 'key': '07e0caf4-1631-47b7-b15f-65515d4c1843',
 'namespace': ['1', 'memories'],
 'created_at': '2024-10-02T17:22:31.590602+00:00',
 'updated_at': '2024-10-02T17:22:31.590605+00:00'}
```

:::

:::js

```typescript
memories[memories.length - 1];
// {
//   value: { food_preference: 'I like pizza' },
//   key: '07e0caf4-1631-47b7-b15f-65515d4c1843',
//   namespace: ['1', 'memories'],
//   createdAt: '2024-10-02T17:22:31.590602+00:00',
//   updatedAt: '2024-10-02T17:22:31.590605+00:00'
// }
```

:::

我们可以访问记忆并在模型调用中使用它们。

:::python

```python
def call_model(state: MessagesState, config: RunnableConfig, *, store: BaseStore):
    # 从配置中获取用户 ID
    user_id = config["configurable"]["user_id"]

    # 对记忆进行命名空间划分
    namespace = (user_id, "memories")

    # 基于最近的消息进行搜索
    memories = store.search(
        namespace,
        query=state["messages"][-1].content,
        limit=3
    )
    info = "\n".join([d.value["memory"] for d in memories])

    # ... 在模型调用中使用记忆
```

:::

:::js

```typescript
const callModel = async (
  state: z.infer<typeof MessagesZodState>,
  config: LangGraphRunnableConfig,
  store: BaseStore
) => {
  // 从配置中获取用户 ID
  const userId = config.configurable?.user_id;

  // 对记忆进行命名空间划分
  const namespace = [userId, "memories"];

  // 基于最近的消息进行搜索
  const memories = await store.search(namespace, {
    query: state.messages[state.messages.length - 1].content,
    limit: 3,
  });
  const info = memories.map((d) => d.value.memory).join("\n");

  // ... 在模型调用中使用记忆
};
```

:::

如果我们创建一个新线程，只要 `user_id` 相同，我们仍然可以访问相同的记忆。

:::python

```python
# 调用图
config = {"configurable": {"thread_id": "2", "user_id": "1"}}

# 让我们再次打个招呼
for update in graph.stream(
    {"messages": [{"role": "user", "content": "hi, tell me about my memories"}]}, config, stream_mode="updates"
):
    print(update)
```

:::

:::js

```typescript
// 调用图
const config = { configurable: { thread_id: "2", user_id: "1" } };

// 让我们再次打个招呼
for await (const update of await graph.stream(
  { messages: [{ role: "user", content: "hi, tell me about my memories" }] },
  { ...config, streamMode: "updates" }
)) {
  console.log(update);
}
```

:::

当我们在 LangGraph Platform 上本地运行（例如，在 LangGraph Studio 中）或使用 LangGraph Platform 时，基础存储是默认可用的，无需在图编译时指定。但是，要启用语义搜索，您**确实**需要在 `langgraph.json` 文件中配置索引设置。例如：

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embeddings-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

有关更多详细信息和配置选项，请参阅[部署指南](../cloud/deployment/semantic_search.md)。

## Checkpointer 库

底层检查点功能由符合 @[BaseCheckpointSaver] 接口的 checkpointer 对象提供支持。LangGraph 提供了几个 checkpointer 实现，它们都通过独立的、可安装的库实现：

:::python

- `langgraph-checkpoint`：checkpointer saver 的基本接口（@[BaseCheckpointSaver]）和序列化/反序列化接口（@[SerializerProtocol][SerializerProtocol]）。包括用于实验的内存 checkpointer 实现（@[InMemorySaver][InMemorySaver]）。LangGraph 自带 `langgraph-checkpoint`。
- `langgraph-checkpoint-sqlite`：使用 SQLite 数据库（@[SqliteSaver][SqliteSaver] / @[AsyncSqliteSaver]）的 LangGraph checkpointer 实现。非常适合实验和本地工作流。需要单独安装。
- `langgraph-checkpoint-postgres`：使用 Postgres 数据库（@[PostgresSaver][PostgresSaver] / @[AsyncPostgresSaver]）的高级 checkpointer，用于 LangGraph Platform。非常适合生产使用。需要单独安装。

:::

:::js

- `@langchain/langgraph-checkpoint`：checkpointer saver 的基本接口（@[BaseCheckpointSaver][BaseCheckpointSaver]）和序列化/反序列化接口（@[SerializerProtocol][SerializerProtocol]）。包括用于实验的内存 checkpointer 实现（@[MemorySaver]）。LangGraph 自带 `@langchain/langgraph-checkpoint`。
- `@langchain/langgraph-checkpoint-sqlite`：使用 SQLite 数据库（@[SqliteSaver]）的 LangGraph checkpointer 实现。非常适合实验和本地工作流。需要单独安装。
- `@langchain/langgraph-checkpoint-postgres`：使用 Postgres 数据库（@[PostgresSaver]）的高级 checkpointer，用于 LangGraph Platform。非常适合生产使用。需要单独安装。

:::

### Checkpointer 接口

:::python
每个 checkpointer 都符合 @[BaseCheckpointSaver] 接口并实现以下方法：

- `.put` - 使用其配置和元数据存储一个检查点。
- `.put_writes` - 存储与检查点关联的中间写入（即 [待处理写入](#pending-writes)）。
- `.get_tuple` - 使用给定的配置（`thread_id` 和 `checkpoint_id`）获取一个检查点元组。这用于填充 `graph.get_state()` 中的 `StateSnapshot`。
- `.list` - 列出与给定配置和筛选条件匹配的检查点。这用于填充 `graph.get_state_history()` 中的状态历史。

如果 checkpointer 与异步图执行一起使用（即通过 `.ainvoke`、`.astream`、`.abatch` 执行图），则将使用上述方法的异步版本（`.aput`、`.aput_writes`、`.aget_tuple`、`.alist`）。

!!! note

    要异步运行您的图，可以使用 `InMemorySaver`，或者 Sqlite/Postgres checkpointers 的异步版本——`AsyncSqliteSaver` / `AsyncPostgresSaver` checkpointers。

:::

:::js
每个 checkpointer 都符合 @[BaseCheckpointSaver][BaseCheckpointSaver] 接口并实现以下方法：

- `.put` - 使用其配置和元数据存储一个检查点。
- `.putWrites` - 存储与检查点关联的中间写入（即 [待处理写入](#pending-writes)）。
- `.getTuple` - 使用给定的配置（`thread_id` 和 `checkpoint_id`）获取一个检查点元组。这用于填充 `graph.getState()` 中的 `StateSnapshot`。
- `.list` - 列出与给定配置和筛选条件匹配的检查点。这用于填充 `graph.getStateHistory()` 中的状态历史。
  :::

### 序列化器 (Serializer)

当 checkpointers 保存图状态时，它们需要序列化状态中的通道值。这是使用序列化器（serializer）对象完成的。

:::python
`langgraph_checkpoint` 定义了 @[protocol][SerializerProtocol] 来实现序列化器，并提供了一个默认实现（@[JsonPlusSerializer][JsonPlusSerializer]），它可以处理多种类型的对象，包括 LangChain 和 LangGraph 的原生类型、日期时间、枚举等。

#### 使用 `pickle` 进行序列化

默认序列化器 @[`JsonPlusSerializer`][JsonPlusSerializer] 在底层使用 ormsgpack 和 JSON，这并不适用于所有类型的对象。

如果您想为我们 msgpack 编码器不直接支持的对象（例如 Pandas 数据帧）提供回退到 pickle 的选项，
您可以在 `JsonPlusSerializer` 的 `pickle_fallback` 参数中使用它：

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.checkpoint.serde.jsonplus import JsonPlusSerializer

# ... 定义图 ...
graph.compile(
    checkpointer=InMemorySaver(serde=JsonPlusSerializer(pickle_fallback=True))
)
```

#### 加密

Checkpointers 可以选择性地加密所有持久化的状态。要启用此功能，请将 @[`EncryptedSerializer`][EncryptedSerializer] 实例传递给任何 `BaseCheckpointSaver` 实现的 `serde` 参数。创建加密序列化程序的最简单方法是通过 @[`from_pycryptodome_aes`][from_pycryptodome_aes]，它从 `LANGGRAPH_AES_KEY` 环境变量读取 AES 密钥（或接受 `key` 参数）：

```python
import sqlite3

from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.sqlite import SqliteSaver

serde = EncryptedSerializer.from_pycryptodome_aes()  # 读取 LANGGRAPH_AES_KEY
checkpointer = SqliteSaver(sqlite3.connect("checkpoint.db"), serde=serde)
```

```python
from langgraph.checkpoint.serde.encrypted import EncryptedSerializer
from langgraph.checkpoint.postgres import PostgresSaver

serde = EncryptedSerializer.from_pycryptodome_aes()
checkpointer = PostgresSaver.from_conn_string("postgresql://...", serde=serde)
checkpointer.setup()
```

在 LangGraph Platform 上运行时，只要 `LANGGRAPH_AES_KEY` 存在，就会自动启用加密，因此您只需提供环境变量即可。其他加密方案可以通过实现 @[`CipherProtocol`][CipherProtocol] 并将其提供给 `EncryptedSerializer` 来使用。
:::

:::js
`@langchain/langgraph-checkpoint` 定义了实现序列化器的协议，并提供了一个默认实现，该实现可以处理多种类型的对象，包括 LangChain 和 LangGraph 的原生类型、日期时间、枚举等。
:::

## 功能

### 人工干预 (Human-in-the-loop)

首先，checkpointers 通过允许人类检查、中断和批准图的步骤来促进[人工干预工作流](agentic_concepts.md#human-in-the-loop)。这些工作流需要 checkpointers，因为人类必须能够查看图在任何时间点的状态，并且图必须能够在人类对状态进行任何更新后恢复执行。请参阅[操作指南](../how-tos/human_in_the_loop/add-human-in-the-loop.md)了解示例。

### 记忆 (Memory)

其次，checkpointers 允许在交互之间进行["记忆"](../concepts/memory.md)。在重复的人工交互（如对话）的情况下，任何后续消息都可以发送到该线程，该线程将保留其对先前交互的记忆。有关如何使用 checkpointers 添加和管理对话记忆的信息，请参阅[添加记忆](../how-tos/memory/add-memory.md)。

### 时间旅行 (Time Travel)

第三，checkpointers 允许["时间旅行"](time-travel.md)，使用户能够回放先前的图执行以审查和/或调试特定的图步骤。此外，checkpointers 使得可以在任意检查点处分支图状态以探索替代轨迹成为可能。

### 容错 (Fault-tolerance)

最后，检查点还提供了容错和错误恢复功能：如果一个或多个节点在给定的超级步中失败，您可以从上一个成功的步骤恢复您的图。此外，当图节点在给定超级步的执行过程中失败时，LangGraph 会存储该超级步中任何其他已成功完成的节点的待处理检查点写入，这样当我们从该超级步恢复图执行时，就不会重新运行成功的节点。

#### 待处理写入

此外，当图节点在给定超级步的执行过程中失败时，LangGraph 会存储该超级步中任何其他已成功完成节点的待处理检查点写入，这样当我们从该超级步恢复图执行时，就不会重新运行成功的节点。