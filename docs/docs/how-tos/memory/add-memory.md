# 添加和管理记忆

AI 应用程序需要[记忆](../../concepts/memory.md)在多次交互中共享上下文。在 LangGraph 中，您可以添加两种类型的记忆：

- [添加短期记忆](#add-short-term-memory) 作为代理[状态](../../concepts/low_level.md#state)的一部分，以支持多轮对话。
- [添加长期记忆](#add-long-term-memory) 以在会话中存储用户特定或应用程序级别的数据。

## 添加短期记忆

**短期**记忆（线程级别[持久化](../../concepts/persistence.md)）使代理能够跟踪多轮对话。要添加短期记忆：

:::python

```python
# highlight-next-line
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.graph import StateGraph

# highlight-next-line
checkpointer = InMemorySaver()

builder = StateGraph(...)
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)

graph.invoke(
    {"messages": [{"role": "user", "content": "hi! i am Bob"}]},
    # highlight-next-line
    {"configurable": {"thread_id": "1"}},
)
```

:::

:::js

```typescript
import { MemorySaver, StateGraph } from "@langchain/langgraph";

const checkpointer = new MemorySaver();

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });

await graph.invoke(
  { messages: [{ role: "user", content: "hi! i am Bob" }] },
  { configurable: { thread_id: "1" } }
);
```

:::

### 生产环境使用

在生产环境中，请使用由数据库支持的 checkpointer：

:::python

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(checkpointer=checkpointer)
```

:::

:::js

```typescript
import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const checkpointer = PostgresSaver.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ checkpointer });
```

:::

??? example "示例：使用 Postgres checkpointer"

    :::python
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres checkpointer 时需要调用 `checkpointer.setup()`

    === "同步"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.postgres import PostgresSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
        # highlight-next-line
        with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "异步"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
        # highlight-next-line
        async with AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer:
            # await checkpointer.setup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```
    :::

    :::js
    ```
    npm install @langchain/langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres checkpointer 时需要调用 `checkpointer.setup()`

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, MessagesZodState, START } from "@langchain/langgraph";
    import { PostgresSaver } from "@langchain/langgraph-checkpoint-postgres";

    const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });

    const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
    const checkpointer = PostgresSaver.fromConnString(DB_URI);
    // await checkpointer.setup();

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", async (state) => {
        const response = await model.invoke(state.messages);
        return { messages: [response] };
      })
      .addEdge(START, "call_model");

    const graph = builder.compile({ checkpointer });

    const config = {
      configurable: {
        thread_id: "1"
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "hi! I'm bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "what's my name?" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }
    ```
    :::

:::python
??? example "示例：使用 [MongoDB](https://pypi.org/project/langgraph-checkpoint-mongodb/) checkpointer"

    ```
    pip install -U pymongo langgraph langgraph-checkpoint-mongodb
    ```

    !!! note "Setup"

        要使用 MongoDB checkpointer，您需要一个 MongoDB 群集。如果还没有群集，请遵循[此指南](https://www.mongodb.com/docs/guides/atlas/cluster/)创建一个群集。

    === "同步"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.mongodb import MongoDBSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "localhost:27017"
        # highlight-next-line
        with MongoDBSaver.from_conn_string(DB_URI) as checkpointer:

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "异步"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.mongodb.aio import AsyncMongoDBSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "localhost:27017"
        # highlight-next-line
        async with AsyncMongoDBSaver.from_conn_string(DB_URI) as checkpointer:

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

??? example "示例：使用 [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) checkpointer"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        首次使用 Redis checkpointer 时需要调用 `checkpointer.setup()`


    === "同步"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.redis import RedisSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"
        # highlight-next-line
        with RedisSaver.from_conn_string(DB_URI) as checkpointer:
            # checkpointer.setup()

            def call_model(state: MessagesState):
                response = model.invoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "异步"

        ```python
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        # highlight-next-line
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"
        # highlight-next-line
        async with AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer:
            # await checkpointer.asetup()

            async def call_model(state: MessagesState):
                response = await model.ainvoke(state["messages"])
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            # highlight-next-line
            graph = builder.compile(checkpointer=checkpointer)

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1"
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what's my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

:::

### 在子图中应用

如果图包含[子图](../../concepts/subgraphs.md)，您只需在编译父图时提供 checkpointer。LangGraph 会自动将 checkpointer 传播到子图。

:::python

```python
from langgraph.graph import START, StateGraph
from langgraph.checkpoint.memory import InMemorySaver
from typing import TypedDict

class State(TypedDict):
    foo: str

# 子图

def subgraph_node_1(state: State):
    return {"foo": state["foo"] + "bar"}

subgraph_builder = StateGraph(State)
subgraph_builder.add_node(subgraph_node_1)
subgraph_builder.add_edge(START, "subgraph_node_1")
# highlight-next-line
subgraph = subgraph_builder.compile()

# 父图

builder = StateGraph(State)
# highlight-next-line
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)
```

:::

:::js

```typescript
import { StateGraph, START, MemorySaver } from "@langchain/langgraph";
import { z } from "zod";

const State = z.object({ foo: z.string() });

const subgraphBuilder = new StateGraph(State)
  .addNode("subgraph_node_1", (state) => {
    return { foo: state.foo + "bar" };
  })
  .addEdge(START, "subgraph_node_1");
const subgraph = subgraphBuilder.compile();

const builder = new StateGraph(State)
  .addNode("node_1", subgraph)
  .addEdge(START, "node_1");

const checkpointer = new MemorySaver();
const graph = builder.compile({ checkpointer });
```

:::

如果您希望子图拥有自己的记忆，则可以使用相应的 checkpointer 选项来编译它。这在[多代理](../../concepts/multi_agent.md)系统中很有用，如果您想让代理跟踪其内部消息历史记录。

:::python

```python
subgraph_builder = StateGraph(...)
# highlight-next-line
subgraph = subgraph_builder.compile(checkpointer=True)
```

:::

:::js

```typescript
const subgraphBuilder = new StateGraph(...);
// highlight-next-line
const subgraph = subgraphBuilder.compile({ checkpointer: true });
```

:::

### 在工具中读取短期记忆 { #read-short-term }

LangGraph 允许代理在其工具中访问其短期记忆（状态）。

:::python

```python
from typing import Annotated
from langgraph.prebuilt import InjectedState, create_react_agent

class CustomState(AgentState):
    # highlight-next-line
    user_id: str

def get_user_info(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """查找用户信息。"""
    # highlight-next-line
    user_id = state["user_id"]
    return "User is John Smith" if user_id == "user_123" else "Unknown user"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_info],
    # highlight-next-line
    state_schema=CustomState,
)

agent.invoke({
    "messages": "look up user information",
    # highlight-next-line
    "user_id": "user_123"
})
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import {
  MessagesZodState,
  LangGraphRunnableConfig,
} from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const CustomState = z.object({
  messages: MessagesZodState.shape.messages,
  userId: z.string(),
});

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config.configurable?.userId;
    return userId === "user_123" ? "User is John Smith" : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getUserInfo],
  stateSchema: CustomState,
});

await agent.invoke(
  { messages: [{ role: "user", content: "look up user information" }] },
  { configurable: { userId: "user_123" } }
);
```

:::

请参阅[上下文](../../agents/context.md)指南了解更多信息。

### 从工具写入短期记忆 { #write-short-term }

要修改执行期间代理的短期记忆（状态），您可以直接从工具返回状态更新。这对于持久化中间结果或使信息可供后续工具或提示访问非常有用。

:::python

```python
from typing import Annotated
from langchain_core.tools import InjectedToolCallId
from langchain_core.runnables import RunnableConfig
from langchain_core.messages import ToolMessage
from langgraph.prebuilt import InjectedState, create_react_agent
from langgraph.prebuilt.chat_agent_executor import AgentState
from langgraph.types import Command

class CustomState(AgentState):
    # highlight-next-line
    user_name: str

def update_user_info(
    tool_call_id: Annotated[str, InjectedToolCallId],
    config: RunnableConfig
) -> Command:
    """查找并更新用户信息。"""
    user_id = config["configurable"].get("user_id")
    name = "John Smith" if user_id == "user_123" else "Unknown user"
    # highlight-next-line
    return Command(update={
        # highlight-next-line
        "user_name": name,
        # 更新消息历史记录
        "messages": [
            ToolMessage(
                "Successfully looked up user information",
                tool_call_id=tool_call_id
            )
        ]
    })

def greet(
    # highlight-next-line
    state: Annotated[CustomState, InjectedState]
) -> str:
    """在找到用户信息后用于问候用户的工具。"""
    user_name = state["user_name"]
    return f"Hello {user_name}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[update_user_info, greet],
    # highlight-next-line
    state_schema=CustomState
)

agent.invoke(
    {"messages": [{"role": "user", "content": "greet the user"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import {
  MessagesZodState,
  LangGraphRunnableConfig,
  Command,
} from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const CustomState = z.object({
  messages: MessagesZodState.shape.messages,
  userName: z.string().optional(),
});

const updateUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userId = config.configurable?.userId;
    const name = userId === "user_123" ? "John Smith" : "Unknown user";
    return new Command({
      update: {
        userName: name,
        // 更新消息历史记录
        messages: [
          {
            role: "tool",
            content: "Successfully looked up user information",
            tool_call_id: config.toolCall?.id,
          },
        ],
      },
    });
  },
  {
    name: "update_user_info",
    description: "Look up and update user info.",
    schema: z.object({}),
  }
);

const greet = tool(
  async (_, config: LangGraphRunnableConfig) => {
    const userName = config.configurable?.userName;
    return `Hello ${userName}!`;
  },
  {
    name: "greet",
    description: "Use this to greet the user once you found their info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [updateUserInfo, greet],
  stateSchema: CustomState,
});

await agent.invoke(
  { messages: [{ role: "user", content: "greet the user" }] },
  { configurable: { userId: "user_123" } }
);
```

:::

## 添加长期记忆

使用长期记忆在会话期间存储用户特定或应用程序特定的数据。

:::python

```python
# highlight-next-line
from langgraph.store.memory import InMemoryStore
from langgraph.graph import StateGraph

# highlight-next-line
store = InMemoryStore()

builder = StateGraph(...)
# highlight-next-line
graph = builder.compile(store=store)
```

:::

:::js

```typescript
import { InMemoryStore, StateGraph } from "@langchain/langgraph";

const store = new InMemoryStore();

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

:::

### 生产环境使用

在生产环境中，请使用由数据库支持的 store：

:::python

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresStore.from_conn_string(DB_URI) as store:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(store=store)
```

:::

:::js

```typescript
import { PostgresStore } from "@langchain/langgraph-checkpoint-postgres";

const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";
const store = PostgresStore.fromConnString(DB_URI);

const builder = new StateGraph(...);
const graph = builder.compile({ store });
```

:::

??? example "示例：使用 Postgres store"

    :::python
    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres store 时需要调用 `store.setup()`

    === "同步"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres import PostgresSaver
        # highlight-next-line
        from langgraph.store.postgres import PostgresStore
        from langgraph.store.base import BaseStore
        import uuid

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

        with (
            # highlight-next-line
            PostgresStore.from_conn_string(DB_URI) as store,
            PostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # store.setup()
            # checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = store.search(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # 如果用户要求模型记住，则存储新记忆
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "异步"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres.aio import AsyncPostgresSaver
        # highlight-next-line
        from langgraph.store.postgres.aio import AsyncPostgresStore
        from langgraph.store.base import BaseStore
        import uuid

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"

        async with (
            # highlight-next-line
            AsyncPostgresStore.from_conn_string(DB_URI) as store,
            AsyncPostgresSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.setup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # 如果用户要求模型记住，则存储新记忆
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```
    :::

    :::js
    ```
    npm install @langchain/langgraph-checkpoint-postgres
    ```

    !!! Setup
        首次使用 Postgres store 时需要调用 `store.setup()`

    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, MessagesZodState, START, LangGraphRunnableConfig } from "@langchain/langgraph";
    import { PostgresSaver, PostgresStore } from "@langchain/langgraph-checkpoint-postgres";
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";

    const model = new ChatAnthropic({ model: "claude-3-5-haiku-20241022" });

    const DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable";

    const store = PostgresStore.fromConnString(DB_URI);
    const checkpointer = PostgresSaver.fromConnString(DB_URI);
    // await store.setup();
    // await checkpointer.setup();

    const callModel = async (
      state: z.infer<typeof MessagesZodState>,
      config: LangGraphRunnableConfig,
    ) => {
      const userId = config.configurable?.userId;
      const namespace = ["memories", userId];
      const memories = await config.store?.search(namespace, { query: state.messages.at(-1)?.content });
      const info = memories?.map(d => d.value.data).join("\n") || "";
      const systemMsg = `You are a helpful assistant talking to the user. User info: ${info}`;

      // 如果用户要求模型记住，则存储新记忆
      const lastMessage = state.messages.at(-1);
      if (lastMessage?.content?.toLowerCase().includes("remember")) {
        const memory = "User name is Bob";
        await config.store?.put(namespace, uuidv4(), { data: memory });
      }

      const response = await model.invoke([
        { role: "system", content: systemMsg },
        ...state.messages
      ]);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addEdge(START, "call_model");

    const graph = builder.compile({
      checkpointer,
      store,
    });

    const config = {
      configurable: {
        thread_id: "1",
        userId: "1",
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "Hi! Remember: my name is Bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }

    const config2 = {
      configurable: {
        thread_id: "2",
        userId: "1",
      }
    };

    for await (const chunk of await graph.stream(
      { messages: [{ role: "user", content: "what is my name?" }] },
      { ...config2, streamMode: "values" }
    )) {
      console.log(chunk.messages.at(-1)?.content);
    }
    ```
    :::

:::python
??? example "示例：使用 [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) store"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        首次使用 Redis store 时需要调用 `store.setup()`


    === "同步"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis import RedisSaver
        # highlight-next-line
        from langgraph.store.redis import RedisStore
        from langgraph.store.base import BaseStore
        import uuid

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"

        with (
            # highlight-next-line
            RedisStore.from_conn_string(DB_URI) as store,
            RedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            store.setup()
            checkpointer.setup()

            def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = store.search(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # 如果用户要求模型记住，则存储新记忆
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    store.put(namespace, str(uuid.uuid4()), {"data": memory})

                response = model.invoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            for chunk in graph.stream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

    === "异步"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis.aio import AsyncRedisSaver
        # highlight-next-line
        from langgraph.store.redis.aio import AsyncRedisStore
        from langgraph.store.base import BaseStore
        import uuid

        model = init_chat_model(model="anthropic:claude-3-5-haiku-latest")

        DB_URI = "redis://localhost:6379"

        async with (
            # highlight-next-line
            AsyncRedisStore.from_conn_string(DB_URI) as store,
            AsyncRedisSaver.from_conn_string(DB_URI) as checkpointer,
        ):
            # await store.setup()
            # await checkpointer.asetup()

            async def call_model(
                state: MessagesState,
                config: RunnableConfig,
                *,
                # highlight-next-line
                store: BaseStore,
            ):
                user_id = config["configurable"]["user_id"]
                namespace = ("memories", user_id)
                # highlight-next-line
                memories = await store.asearch(namespace, query=str(state["messages"][-1].content))
                info = "\n".join([d.value["data"] for d in memories])
                system_msg = f"You are a helpful assistant talking to the user. User info: {info}"

                # 如果用户要求模型记住，则存储新记忆
                last_message = state["messages"][-1]
                if "remember" in last_message.content.lower():
                    memory = "User name is Bob"
                    # highlight-next-line
                    await store.aput(namespace, str(uuid.uuid4()), {"data": memory})

                response = await model.ainvoke(
                    [{"role": "system", "content": system_msg}] + state["messages"]
                )
                return {"messages": response}

            builder = StateGraph(MessagesState)
            builder.add_node(call_model)
            builder.add_edge(START, "call_model")

            graph = builder.compile(
                checkpointer=checkpointer,
                # highlight-next-line
                store=store,
            )

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "1",
                    # highlight-next-line
                    "user_id": "1",
                }
            }
            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "Hi! Remember: my name is Bob"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()

            config = {
                "configurable": {
                    # highlight-next-line
                    "thread_id": "2",
                    "user_id": "1",
                }
            }

            async for chunk in graph.astream(
                {"messages": [{"role": "user", "content": "what is my name?"}]},
                # highlight-next-line
                config,
                stream_mode="values",
            ):
                chunk["messages"][-1].pretty_print()
        ```

:::

### 在工具中读取长期记忆 { #read-long-term }

:::python

```python title="代理可用于查找用户信息的一个工具"
from langchain_core.runnables import RunnableConfig
from langgraph.config import get_store
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore

# highlight-next-line
store = InMemoryStore() # (1)!

# highlight-next-line
store.put(  # (2)!
    ("users",),  # (3)!
    "user_123",  # (4)!
    {
        "name": "John Smith",
        "language": "English",
    } # (5)!
)

def get_user_info(config: RunnableConfig) -> str:
    """查找用户信息。"""
    # 与传递给 `create_react_agent` 的内容相同
    # highlight-next-line
    store = get_store() # (6)!
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    user_info = store.get(("users",), user_id) # (7)!
    return str(user_info.value) if user_info else "Unknown user"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_user_info],
    # highlight-next-line
    store=store # (8)!
)

# 运行代理
agent.invoke(
    {"messages": [{"role": "user", "content": "look up user information"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}}
)
```

1. `InMemoryStore` 是一个将数据存储在内存中的 store。在生产环境中，通常会使用数据库或其他持久存储。请查阅[store 文档](../../reference/store.md)了解更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产级别的 store。
2. 在此示例中，我们使用 `put` 方法向 store 写入一些示例数据。更多详细信息请参阅 @[BaseStore.put] API 参考。
3. 第一个参数是 namespace。它用于将相关数据分组。在此示例中，我们使用 `users` namespace 来分组用户数据。
4. namespace 内的一个键。此示例使用用户 ID 作为键。
5. 要为给定用户存储的数据。
6. `get_store` 函数用于访问 store。您可以从代码中的任何位置调用它，包括工具和提示。此函数返回创建代理时传递给代理的 store。
7. `get` 方法用于从 store 中检索数据。第一个参数是 namespace，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关该值的一些元数据。
8. `store` 被传递给代理。这使得代理在运行工具时可以访问 store。您还可以使用 config 中的 store 从代码中的任何位置访问它。
   :::

:::js

```typescript title="代理可用于查找用户信息的一个工具"
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { LangGraphRunnableConfig, InMemoryStore } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const store = new InMemoryStore(); // (1)!

await store.put(
  // (2)!
  ["users"], // (3)!
  "user_123", // (4)!
  {
    name: "John Smith",
    language: "English",
  } // (5)!
);

const getUserInfo = tool(
  async (_, config: LangGraphRunnableConfig) => {
    /**Look up user info.*/
    // 与传递给 `createReactAgent` 的内容相同
    const store = config.store; // (6)!
    const userId = config.configurable?.userId;
    const userInfo = await store?.get(["users"], userId); // (7)!
    return userInfo?.value ? JSON.stringify(userInfo.value) : "Unknown user";
  },
  {
    name: "get_user_info",
    description: "Look up user info.",
    schema: z.object({}),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [getUserInfo],
  store, // (8)!
});

// 运行代理
await agent.invoke(
  { messages: [{ role: "user", content: "look up user information" }] },
  { configurable: { userId: "user_123" } }
);
```

1. `InMemoryStore` 是一个将数据存储在内存中的 store。在生产环境中，通常会使用数据库或其他持久存储。请查阅[store 文档](../../reference/store.md)了解更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产级别的 store。
2. 在此示例中，我们使用 `put` 方法向 store 写入一些示例数据。更多详细信息请参阅 @[BaseStore.put] API 参考。
3. 第一个参数是 namespace。它用于将相关数据分组。在此示例中，我们使用 `users` namespace 来分组用户数据。
4. namespace 内的一个键。此示例使用用户 ID 作为键。
5. 要为给定用户存储的数据。
6. store 可通过 config 访问。您可以从代码中的任何位置调用它，包括工具和提示。此函数返回创建代理时传递给代理的 store。
7. `get` 方法用于从 store 中检索数据。第一个参数是 namespace，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关该值的一些元数据。
8. `store` 被传递给代理。这使得代理在运行工具时可以访问 store。您还可以使用 config 中的 store 从代码中的任何位置访问它。
   :::

### 从工具写入长期记忆 { #write-long-term }

:::python

```python title="用于更新用户信息的工具示例"
from typing_extensions import TypedDict

from langgraph.config import get_store
from langchain_core.runnables import RunnableConfig
from langgraph.prebuilt import create_react_agent
from langgraph.store.memory import InMemoryStore

store = InMemoryStore() # (1)!

class UserInfo(TypedDict): # (2)!
    name: str

def save_user_info(user_info: UserInfo, config: RunnableConfig) -> str: # (3)!
    """保存用户信息。"""
    # 与传递给 `create_react_agent` 的内容相同
    # highlight-next-line
    store = get_store() # (4)!
    user_id = config["configurable"].get("user_id")
    # highlight-next-line
    store.put(("users",), user_id, user_info) # (5)!
    return "Successfully saved user info."

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[save_user_info],
    # highlight-next-line
    store=store
)

# 运行代理
agent.invoke(
    {"messages": [{"role": "user", "content": "My name is John Smith"}]},
    # highlight-next-line
    config={"configurable": {"user_id": "user_123"}} # (6)!
)

# 您可以直接访问 store 来获取值
store.get(("users",), "user_123").value
```

1. `InMemoryStore` 是一个将数据存储在内存中的 store。在生产环境中，通常会使用数据库或其他持久存储。请查阅[store 文档](../../reference/store.md)了解更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产级别的 store。
2. `UserInfo` 类是一个 `TypedDict`，它定义了用户信息结构。LLM 将使用它来根据 schema 格式化响应。
3. `save_user_info` 函数是一个允许代理更新用户信息的工具。这对于聊天应用程序可能很有用，用户希望更新他们的个人资料信息。
4. `get_store` 函数用于访问 store。您可以从代码中的任何位置调用它，包括工具和提示。此函数返回创建代理时传递给代理的 store。
5. `put` 方法用于将数据存储在 store 中。第一个参数是 namespace，第二个参数是键。这将把用户信息存储在 store 中。
6. `user_id` 在 config 中传递。它用于标识正在更新信息的用户的 ID。
   :::

:::js

```typescript title="用于更新用户信息的工具示例"
import { tool } from "@langchain/core/tools";
import { z } from "zod";
import { LangGraphRunnableConfig, InMemoryStore } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";

const store = new InMemoryStore(); // (1)!

const UserInfo = z.object({
  // (2)!
  name: z.string(),
});

const saveUserInfo = tool(
  async (
    userInfo: z.infer<typeof UserInfo>,
    config: LangGraphRunnableConfig
  ) => {
    // (3)!
    /**Save user info.*/
    // 与传递给 `createReactAgent` 的内容相同
    const store = config.store; // (4)!
    const userId = config.configurable?.userId;
    await store?.put(["users"], userId, userInfo); // (5)!
    return "Successfully saved user info.";
  },
  {
    name: "save_user_info",
    description: "Save user info.",
    schema: UserInfo,
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [saveUserInfo],
  store,
});

// 运行代理
await agent.invoke(
  { messages: [{ role: "user", content: "My name is John Smith" }] },
  { configurable: { userId: "user_123" } } // (6)!
);

// 您可以访问 store 直接获取值
const result = await store.get(["users"], "user_123");
console.log(result?.value);
```

1. `InMemoryStore` 是一个将数据存储在内存中的 store。在生产环境中，通常会使用数据库或其他持久存储。请查阅[store 文档](../../reference/store.md)了解更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产级别的 store。
7. `UserInfo` schema 定义了用户信息结构。LLM 将使用它来根据 schema 格式化响应。
8. `saveUserInfo` 函数是一个允许代理更新用户信息的工具。这对于聊天应用程序可能很有用，用户希望更新他们的个人资料信息。
9. store 可通过 config 访问。您可以从代码中的任何位置调用它，包括工具和提示。此函数返回创建代理时传递给代理的 store。
10. `put` 方法用于将数据存储在 store 中。第一个参数是 namespace，第二个参数是键。这将把用户信息存储在 store 中。
11. `userId` 在 config 中传递。它用于标识正在更新信息的用户的 ID。
   :::

### 使用语义搜索

在图的记忆 store 中启用语义搜索，让图代理通过语义相似性搜索 store 中的项目。

:::python

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# 启用语义搜索的 Store 的创建
embeddings = init_embeddings("openai:text-embedding-3-small")
store = InMemoryStore(
    index={
        "embed": embeddings,
        "dims": 1536,
    }
)

store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

items = store.search(
    ("user_123", "memories"), query="I'm hungry", limit=1
)
```

:::

:::js

```typescript
import { OpenAIEmbeddings } from "@langchain/openai";
import { InMemoryStore } from "@langchain/langgraph";

// 启用语义搜索的 Store 的创建
const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
const store = new InMemoryStore({
  index: {
    embeddings,
    dims: 1536,
  },
});

await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

const items = await store.search(["user_123", "memories"], {
  query: "I'm hungry",
  limit: 1,
});
```

:::

??? example "长期记忆与语义搜索"

    :::python
    ```python
    from typing import Optional

    from langchain.embeddings import init_embeddings
    from langchain.chat_models import init_chat_model
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore
    from langgraph.graph import START, MessagesState, StateGraph
    from langchain_core.messages import HumanMessage

    llm = init_chat_model("openai:gpt-4o-mini")

    # 启用语义搜索的 Store 的创建
    embeddings = init_embeddings("openai:text-embedding-3-small")
    store = InMemoryStore(
        index={
            "embed": embeddings,
            "dims": 1536,
        }
    )

    store.put(("user_123", "memories"), "1", {"text": "I love pizza"})
    store.put(("user_123", "memories"), "2", {"text": "I am a plumber"})

    def chat(state, *, store: BaseStore):
        # 基于用户最近一条消息进行搜索
        items = store.search(
            ("user_123", "memories"), query=state["messages"][-1].content, limit=2
        )
        memories = "\n".join(item.value["text"] for item in items)
        memories = f"## Memories of user\n{memories}" if memories else ""
        response = llm.invoke(
            [
                {"role": "system", "content": f"You are a helpful assistant.\n{memories}"},
                *state["messages"],
            ]
        )
        return {"messages": [response]}


    builder = StateGraph(MessagesState)
    builder.add_node(chat)
    builder.add_edge(START, "chat")
    graph = builder.compile(store=store)

    for message, metadata in graph.stream(
        input={"messages": [{"role": "user", "content": "I'm hungry"}]},
        stream_mode="messages",
    ):
        print(message.content, end="")
    ```
    :::

    :::js
    ```typescript
    import { OpenAIEmbeddings, ChatOpenAI } from "@langchain/openai";
    import { StateGraph, START, MessagesZodState, InMemoryStore } from "@langchain/langgraph";
    import { z } from "zod";
    import { HumanMessage } from "@langchain/core/messages";

    const llm = new ChatOpenAI({ model: "gpt-4o-mini" });

    // 启用语义搜索的 Store 的创建
    const embeddings = new OpenAIEmbeddings({ model: "text-embedding-3-small" });
    const store = new InMemoryStore({
      index: {
        embeddings,
        dims: 1536,
      }
    });

    await store.put(["user_123", "memories"], "1", { text: "I love pizza" });
    await store.put(["user_123", "memories"], "2", { text: "I am a plumber" });

    const chat = async (state: z.infer<typeof MessagesZodState>, config) => {
      // 基于用户最近一条消息进行搜索
      const items = await config.store.search(
        ["user_123", "memories"],
        { query: state.messages.at(-1)?.content, limit: 2 }
      );
      const memories = items.map(item => item.value.text).join("\n");
      const memoriesText = memories ? `## Memories of user\n${memories}` : "";

      const response = await llm.invoke([
        { role: "system", content: `You are a helpful assistant.\n${memoriesText}` },
        ...state.messages,
      ]);

      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("chat", chat)
      .addEdge(START, "chat");
    const graph = builder.compile({ store });

    for await (const [message, metadata] of await graph.stream(
      { messages: [{ role: "user", content: "I'm hungry" }] },
      { streamMode: "messages" }
    )) {
      if (message.content) {
        console.log(message.content);
      }
    }
    ```
    :::

请参阅[本指南](../../cloud/deployment/semantic_search.md)，了解有关如何将语义搜索与 LangGraph 记忆 store 结合使用的更多信息。

## 管理短期记忆

启用[短期记忆](#add-short-term-memory)后，长时间的对话可能会超出 LLM 的上下文窗口。常见的解决方案包括：

- [修剪消息](#trim-messages)：删除前 N 条或最后 N 条消息（在调用 LLM 之前）
- [删除消息](#delete-messages)：永久从 LangGraph 状态中删除
- [汇总消息](#summarize-messages)：汇总历史记录中的较早消息，并用摘要替换它们
- [管理检查点](#manage-checkpoints) 以存储和检索消息历史记录
- 自定义策略（例如，消息过滤等）

这使代理能够在不超出 LLM 的上下文窗口的情况下跟踪对话。

### 修剪消息

大多数 LLM 都有支持的最大上下文窗口（以 token 为单位）。决定何时截断消息的一种方法是对消息历史记录中的 token 进行计数，并在接近该限制时进行截断。如果您使用的是 LangChain，可以使用修剪消息实用程序，并指定要保留的 token 数量，以及用于处理边界的 `strategy`（例如，保留最后一个 `maxTokens`）。

=== "在代理中"

    :::python
    要在代理中修剪消息历史记录，请使用 @[`pre_model_hook`][create_react_agent] 和 [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) 函数：

    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )
    from langgraph.prebuilt import create_react_agent

    # 每次调用 LLM 的节点之前都会调用此函数
    def pre_model_hook(state):
        trimmed_messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=384,
            start_on="human",
            end_on=("human", "tool"),
        )
        # highlight-next-line
        return {"llm_input_messages": trimmed_messages}

    checkpointer = InMemorySaver()
    agent = create_react_agent(
        model,
        tools,
        # highlight-next-line
        pre_model_hook=pre_model_hook,
        checkpointer=checkpointer,
    )
    ```
    :::

    :::js
    要在代理中修剪消息历史记录，请使用 `stateModifier` 和 [`trimMessages`](https://js.langchain.com/docs/how_to/trim_messages/) 函数：

    ```typescript
    import { trimMessages } from "@langchain/core/messages";
    import { createReactAgent } from "@langchain/langgraph/prebuilt";

    // 每次调用 LLM 的节点之前都会调用此函数
    const stateModifier = async (state) => {
      return trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 384,
        startOn: "human",
        endOn: ["human", "tool"],
      });
    };

    const checkpointer = new MemorySaver();
    const agent = createReactAgent({
      llm: model,
      tools,
      stateModifier,
      checkpointer,
    });
    ```
    :::

=== "在工作流中"

    :::python
    要修剪消息历史记录，请使用 [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) 函数：

    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )

    def call_model(state: MessagesState):
        # highlight-next-line
        messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    ...
    ```
    :::

    :::js
    要修剪消息历史记录，请使用 [`trimMessages`](https://js.langchain.com/docs/how_to/trim_messages/) 函数：

    ```typescript
    import { trimMessages } from "@langchain/core/messages";

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 128,
        startOn: "human",
        endOn: ["human", "tool"],
      });
      const response = await model.invoke(messages);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel);
    // ...
    ```
    :::

??? example "完整示例：修剪消息"

    :::python
    ```python
    # highlight-next-line
    from langchain_core.messages.utils import (
        # highlight-next-line
        trim_messages,
        # highlight-next-line
        count_tokens_approximately
    # highlight-next-line
    )
    from langchain.chat_models import init_chat_model
    from langgraph.graph import StateGraph, START, MessagesState

    model = init_chat_model("anthropic:claude-3-7-sonnet-latest")
    summarization_model = model.bind(max_tokens=128)

    def call_model(state: MessagesState):
        # highlight-next-line
        messages = trim_messages(
            state["messages"],
            strategy="last",
            token_counter=count_tokens_approximately,
            max_tokens=128,
            start_on="human",
            end_on=("human", "tool"),
        )
        response = model.invoke(messages)
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(MessagesState)
    builder.add_node(call_model)
    builder.add_edge(START, "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    ```

    ```
    ================================== Ai Message ==================================

    Your name is Bob, as you mentioned when you first introduced yourself.
    ```
    :::

    :::js
    ```typescript
    import { trimMessages } from "@langchain/core/messages";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, MessagesZodState, MemorySaver } from "@langchain/langgraph";
    import { z } from "zod";

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20241022" });

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const messages = trimMessages(state.messages, {
        strategy: "last",
        maxTokens: 128,
        startOn: "human",
        endOn: ["human", "tool"],
      });
      const response = await model.invoke(messages);
      return { messages: [response] };
    };

    const checkpointer = new MemorySaver();
    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addEdge(START, "call_model");
    const graph = builder.compile({ checkpointer });

    const config = { configurable: { thread_id: "1" } };
    await graph.invoke({ messages: [{ role: "user", content: "hi, my name is bob" }] }, config);
    await graph.invoke({ messages: [{ role: "user", content: "write a short poem about cats" }] }, config);
    await graph.invoke({ messages: [{ role: "user", content: "now do the same but for dogs" }] }, config);
    const finalResponse = await graph.invoke({ messages: [{ role: "user", content: "what's my name?" }] }, config);

    console.log(finalResponse.messages.at(-1)?.content);
    ```

    ```
    Your name is Bob, as you mentioned when you first introduced yourself.
    ```
    :::

### 删除消息

您可以从图状态中删除消息，以管理消息历史记录。当您想要删除特定消息或清空整个消息历史记录时，这很有用。

:::python
要从图状态中删除消息，可以使用 `RemoveMessage`。要使 `RemoveMessage` 生效，您需要使用带有 @[`add_messages`][add_messages] [reducer](../../concepts/low_level.md#reducers) 的状态键，例如 [`MessagesState`](../../concepts/low_level.md#messagesstate)。

要删除特定消息：

```python
# highlight-next-line
from langchain_core.messages import RemoveMessage

def delete_messages(state):
    messages = state["messages"]
    if len(messages) > 2:
        # 删除最早的两条消息
        # highlight-next-line
        return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}
```

要删除**所有**消息：

```python
# highlight-next-line
from langgraph.graph.message import REMOVE_ALL_MESSAGES

def delete_messages(state):
    # highlight-next-line
    return {"messages": [RemoveMessage(id=REMOVE_ALL_MESSAGES)]}
```

:::

:::js
要从图状态中删除消息，可以使用 `RemoveMessage`。要使 `RemoveMessage` 生效，您需要使用带有 @[`messagesStateReducer`][messagesStateReducer] [reducer](../../concepts/low_level.md#reducers) 的状态键，例如 `MessagesZodState`。

要删除特定消息：

```typescript
import { RemoveMessage } from "@langchain/core/messages";

const deleteMessages = (state) => {
  const messages = state.messages;
  if (messages.length > 2) {
    // 删除最早的两条消息
    return {
      messages: messages
        .slice(0, 2)
        .map((m) => new RemoveMessage({ id: m.id })),
    };
  }
};
```

:::

!!! warning

    删除消息时，**请确保**生成的消息历史记录有效。检查您正在使用的 LLM 提供商的限制。例如：

    * 某些提供商期望消息历史记录以 `user` 消息开头
    * 大多数提供商要求包含工具调用的 `assistant` 消息后面必须跟相应的 `tool` 结果消息。

??? example "完整示例：删除消息"

    :::python
    ```python
    # highlight-next-line
    from langchain_core.messages import RemoveMessage

    def delete_messages(state):
        messages = state["messages"]
        if len(messages) > 2:
            # 删除最早的两条消息
            # highlight-next-line
            return {"messages": [RemoveMessage(id=m.id) for m in messages[:2]]}

    def call_model(state: MessagesState):
        response = model.invoke(state["messages"])
        return {"messages": response}

    builder = StateGraph(MessagesState)
    builder.add_sequence([call_model, delete_messages])
    builder.add_edge(START, "call_model")

    checkpointer = InMemorySaver()
    app = builder.compile(checkpointer=checkpointer)

    for event in app.stream(
        {"messages": [{"role": "user", "content": "hi! I'm bob"}]},
        config,
        stream_mode="values"
    ):
        print([(message.type, message.content) for message in event["messages"]])

    for event in app.stream(
        {"messages": [{"role": "user", "content": "what's my name?"}]},
        config,
        stream_mode="values"
    ):
        print([(message.type, message.content) for message in event["messages"]])
    ```

    ```
    [('human', "hi! I'm bob")]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?')]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?")]
    [('human', "hi! I'm bob"), ('ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'), ('human', "what's my name?"), ('ai', 'Your name is Bob.')]
    [('human', "what's my name?"), ('ai', 'Your name is Bob.')]
    ```
    :::

    :::js
    ```typescript
    import { RemoveMessage } from "@langchain/core/messages";
    import { ChatAnthropic } from "@langchain/anthropic";
    import { StateGraph, START, MessagesZodState, MemorySaver } from "@langchain/langgraph";
    import { z } from "zod";

    const model = new ChatAnthropic({ model: "claude-3-5-sonnet-20241022" });

    const deleteMessages = (state: z.infer<typeof MessagesZodState>) => {
      const messages = state.messages;
      if (messages.length > 2) {
        // 删除最早的两条消息
        return { messages: messages.slice(0, 2).map(m => new RemoveMessage({ id: m.id })) };
      }
      return {};
    };

    const callModel = async (state: z.infer<typeof MessagesZodState>) => {
      const response = await model.invoke(state.messages);
      return { messages: [response] };
    };

    const builder = new StateGraph(MessagesZodState)
      .addNode("call_model", callModel)
      .addNode("delete_messages", deleteMessages)
      .addEdge(START, "call_model")
      .addEdge("call_model", "delete_messages");

    const checkpointer = new MemorySaver();
    const app = builder.compile({ checkpointer });

    const config = { configurable: { thread_id: "1" } };

    for await (const event of await app.stream(
      { messages: [{ role: "user", content: "hi! I'm bob" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(event.messages.map(message => [message.getType(), message.content]));
    }

    for await (const event of await app.stream(
      { messages: [{ role: "user", content: "what's my name?" }] },
      { ...config, streamMode: "values" }
    )) {
      console.log(event.messages.map(message => [message.getType(), message.content]));
    }
    ```

    ```
    [['human', "hi! I'm bob"]]
    [['human', "hi! I'm bob"], ['ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?']]
    [['human', "hi! I'm bob"], ['ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'], ['human', "what's my name?"]]
    [['human', "hi! I'm bob"], ['ai', 'Hi Bob! How are you doing today? Is there anything I can help you with?'], ['human', "what's my name?"], ['ai', 'Your name is Bob.']]
    [['human', "what's my name?"], ['ai', 'Your name is Bob.']]
    ```
    :::

### 汇总消息

如上所示，修剪或删除消息的问题在于，您可能会丢失因消息队列筛选而丢失的信息。因此，一些应用程序受益于更复杂的方​​法，即使用聊天模型汇总消息历史记录。

![](../../concepts/img/memory/summary.png)

=== "在代理中"

    :::python
    要在代理中汇总消息历史记录，请使用 @[`pre_model_hook`][create_react_agent] 和预构建的 [`SummarizationNode`](https://langchain-ai.github.io/langmem/reference/short_term/#langmem.short_term.SummarizationNode) 抽象：

    ```python
    from langchain_anthropic import ChatAnthropic
    from langmem.short_term import SummarizationNode, RunningSummary
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.prebuilt import create_react_agent
    from langgraph.prebuilt.chat_agent_executor import AgentState
    from langgraph.checkpoint.memory import InMemorySaver
    from typing import Any

    model = ChatAnthropic(model="claude-3-7-sonnet-latest")

    summarization_node = SummarizationNode( # (1)!
        token_counter=count_tokens_approximately,
        model=model,
        max_tokens=384,
        max_summary_tokens=128,
        output_messages_key="llm_input_messages",
    )

    class State(AgentState):
        # 注意：我们添加了这个键来跟踪之前的摘要信息
        # 以确保我们不会在每次调用 LLM 时都进行摘要
        # highlight-next-line
        context: dict[str, RunningSummary]  # (2)!


    checkpointer = InMemorySaver() # (3)!

    agent = create_react_agent(
        model=model,
        tools=tools,
        # highlight-next-line
        pre_model_hook=summarization_node, # (4)!
        # highlight-next-line
        state_schema=State, # (5)!
        checkpointer=checkpointer,
    )
    ```

    1. `InMemorySaver` 是一个将代理状态存储在内存中的 checkpointer。在生产环境中，通常会使用数据库或其他持久存储。请查阅[checkpointer 文档](../../reference/checkpoints.md)了解更多选项。如果您使用 **LangGraph Platform** 进行部署，该平台将为您提供生产级别的 checkpointer。
    2. `context` 键已添加到代理的状态中。该键包含用于摘要节点的簿记信息。它用于跟踪最后一个摘要信息，并确保代理不会每次都进行摘要，这可能效率低下。
    3. `checkpointer` 已传递给代理。这使代理能够跨调用持久化其状态。
    4. `pre_model_hook` 设置为 `SummarizationNode`。在将消息历史记录发送到 LLM 之前，此节点将对其进行汇总。摘要节点将自动处理汇总过程并使用新摘要更新代理的状态。如果您愿意，可以替换为自定义实现。请参阅 @[create_react_agent][create_react_agent] API 参考了解更多详细信息。
    5. `state_schema` 设置为 `State` 类，其中包含额外的 `context` 键的自定义状态。
    :::

=== "在工作流中"

    :::python
    可以使用提示和编排逻辑来汇总消息历史记录。例如，在 LangGraph 中，您可以扩展 [`MessagesState`](../../concepts/low_level.md#working-with-messages-in-graph-state) 以包含 `summary` 键：

    ```python
    from langgraph.graph import MessagesState
    class State(MessagesState):
        summary: str
    ```

    然后，您可以使用任何现有摘要作为下一个摘要的上下文来生成聊天历史记录的摘要。此类 `summarize_conversation` 节点可以在 `messages` 状态键中累积一定数量的消息后调用。

    ```python
    def summarize_conversation(state: State):

        # 首先，我们获取任何现有摘要
        summary = state.get("summary", "")

        # 创建我们的摘要提示
        if summary:

            # 摘要已存在
            summary_message = (
                f"This is a summary of the conversation to date: {summary}\n\n"
                "Extend the summary by taking into account the new messages above:"
            )

        else:
            summary_message = "Create a summary of the conversation above:"

        # 将提示添加到我们的历史记录中
        messages = state["messages"] + [HumanMessage(content=summary_message)]
        response = model.invoke(messages)

        # 删除除最后两条消息之外的所有消息
        delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
        return {"summary": response.content, "messages": delete_messages}
    ```
    :::

    :::js
    可以使用提示和编排逻辑来汇总消息历史记录。例如，在 LangGraph 中，您可以扩展 [`MessagesZodState`](../../concepts/low_level.md#working-with-messages-in-graph-state) 以包含 `summary` 键：

    ```typescript
    import { MessagesZodState } from "@langchain/langgraph";
    import { z } from "zod";

    const State = MessagesZodState.merge(z.object({
      summary: z.string().optional(),
    }));
    ```

    然后，您可以使用任何现有摘要作为下一个摘要的上下文来生成聊天历史记录的摘要。此类 `summarizeConversation` 节点可以在 `messages` 状态键中累积一定数量的消息后调用。

    ```typescript
    import { RemoveMessage, HumanMessage } from "@langchain/core/messages";

    const summarizeConversation = async (state: z.infer<typeof State>) => {
      // 首先，我们汇总对话
      const { summary, messages } = state;
      let summaryMessage: string;
      if (summary) {
        // 如果摘要已存在，我们将使用与不存在摘要时不同的系统提示
        // 来汇总它
        summaryMessage =
          `This is a summary of the conversation to date: ${summary}\n\n` +
          "Extend the summary by taking into account the new messages above:";
      } else {
        summaryMessage = "Create a summary of the conversation above:";
      }

      const allMessages = [
        ...messages,
        new HumanMessage({ content: summaryMessage })
      ];
      const response = await model.invoke(allMessages);

      // 我们现在需要删除不再显示的消息
      // 我将删除最后两条消息之外的所有消息，但您可以更改此设置
      const deleteMessages = state.messages
        .slice(0, -2)
        .map(m => new RemoveMessage({ id: m.id }));

      if (typeof response.content !== "string") {
        throw new Error("Expected a string response from the model");
      }

      return {
        summary: response.content,
        messages: deleteMessages
      };
    };
    ```
    :::

??? example "完整示例：汇总消息"

    :::python
    ```python
    from typing import Any, TypedDict

    from langchain.chat_models import init_chat_model
    from langchain_core.messages import AnyMessage
    from langchain_core.messages.utils import count_tokens_approximately
    from langgraph.graph import StateGraph, START, MessagesState
    from langgraph.checkpoint.memory import InMemorySaver
    # highlight-next-line
    from langmem.short_term import SummarizationNode, RunningSummary

    model = init_chat_model("anthropic:claude-3-7-sonnet-latest")
    summarization_model = model.bind(max_tokens=128)

    class State(MessagesState):
        # highlight-next-line
        context: dict[str, RunningSummary]  # (1)!

    class LLMInputState(TypedDict):  # (2)!
        summarized_messages: list[AnyMessage]
        context: dict[str, RunningSummary]

    # highlight-next-line
    summarization_node = SummarizationNode(
        token_counter=count_tokens_approximately,
        model=summarization_model,
        max_tokens=256,
        max_tokens_before_summary=256,
        max_summary_tokens=128,
    )

    # highlight-next-line
    def call_model(state: LLMInputState):  # (3)!
        response = model.invoke(state["summarized_messages"])
        return {"messages": [response]}

    checkpointer = InMemorySaver()
    builder = StateGraph(State)
    builder.add_node(call_model)
    # highlight-next-line
    builder.add_node("summarize", summarization_node)
    builder.add_edge(START, "summarize")
    builder.add_edge("summarize", "call_model")
    graph = builder.compile(checkpointer=checkpointer)

    # 调用图
    config = {"configurable": {"thread_id": "1"}}
    graph.invoke({"messages": "hi, my name is bob"}, config)
    graph.invoke({"messages": "write a short poem about cats"}, config)
    graph.invoke({"messages": "now do the same but for dogs"}, config)
    final_response = graph.invoke({"messages": "what's my name?"}, config)

    final_response["messages"][-1].pretty_print()
    print("\nSummary:", final_response["context"]["running_summary"].summary)
    ```

    1. 我们将在 `context` 字段中跟踪我们正在进行的摘要
    (由 `SummarizationNode` 期望)。
    2. 定义仅用于过滤传递给 `call_model` 节点的输入的私有状态。
    3. 我们在此处传递私有输入状态，以隔离摘要节点返回的消息。

    ```
    ================================== Ai Message ==================================

    From our conversation, I can see that you introduced yourself as Bob. That's the name you shared with me when we began talking.

    Summary: In this conversation, I was introduced to Bob, who then asked me to write a poem about cats. I composed a poem titled "The Mystery of Cats" that captured cats' graceful movements, independent nature, and their special relationship with humans. Bob then requested a similar poem about dogs, so I wrote "The Joy of Dogs," which highlighted dogs' loyalty, enthusiasm, and loving companionship. Both poems were written in a similar style but emphasized the distinct characteristics that make each pet special.
    ```
    :::

    :::js
    ```typescript
    import { ChatAnthropic } from "@langchain/anthropic";
    import {
      SystemMessage,
      HumanMessage,
      RemoveMessage,
      type BaseMessage
    } from "@langchain/core/messages";
    import {
      MessagesZodState,
      StateGraph,
      START,
      END,
      MemorySaver,
    } from "@langchain/langgraph";
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";

    const memory = new MemorySaver();

    // 除了 `messages` 键（MessagesZodState 已有）之外，我们还将添加 `summary` 属性
    const GraphState = z.object({
      messages: MessagesZodState.shape.messages,
      summary: z.string().default(""),
    });

    // 我们将使用此模型来进行对话和汇总
    const model = new ChatAnthropic({ model: "claude-3-haiku-20240307" });

    // 定义调用模型的逻辑
    const callModel = async (state: z.infer<typeof GraphState>) => {
      // 如果存在摘要，我们在其中添加此摘要作为系统消息
      const { summary } = state;
      let { messages } = state;
      if (summary) {
        const systemMessage = new SystemMessage({
          id: uuidv4(),
          content: `Summary of conversation earlier: ${summary}`,
        });
        messages = [systemMessage, ...messages];
      }
      const response = await model.invoke(messages);
      // 我们返回一个对象，因为它将被添加到现有状态中
      return { messages: [response] };
    };

    // 我们现在定义确定何时结束或汇总对话的逻辑
    const shouldContinue = (state: z.infer<typeof GraphState>) => {
      const messages = state.messages;
      // 如果消息超过六条，则汇总对话
      if (messages.length > 6) {
        return "summarize_conversation";
      }
      // 否则就可以结束
      return END;
    };

    const summarizeConversation = async (state: z.infer<typeof GraphState>) => {
      // 首先，我们汇总对话
      const { summary, messages } = state;
      let summaryMessage: string;
      if (summary) {
        // 如果摘要已存在，我们将使用与不存在摘要时不同的系统提示
        // 来汇总它
        summaryMessage =
          `This is a summary of the conversation to date: ${summary}\n\n` +
          "Extend the summary by taking into account the new messages above:";
      } else {
        summaryMessage = "Create a summary of the conversation above:";
      }

      const allMessages = [
        ...messages,
        new HumanMessage({ id: uuidv4(), content: summaryMessage })
      ];

      const response = await model.invoke(allMessages);

      // 我们现在需要删除不再显示的消息
      // 我将删除最后两条消息之外的所有消息，但您可以更改此设置
      const deleteMessages = messages
        .slice(0, -2)
        .map((m) => new RemoveMessage({ id: m.id! }));

      if (typeof response.content !== "string") {
        throw new Error("Expected a string response from the model");
      }

      return { summary: response.content, messages: deleteMessages };
    };

    // 定义一个新图
    const workflow = new StateGraph(GraphState)
      // 定义对话节点和汇总节点
      .addNode("conversation", callModel)
      .addNode("summarize_conversation", summarizeConversation)
      // 将入口点设为对话
      .addEdge(START, "conversation")
      // 我们现在添加一个条件边
      .addConditionalEdges(
        // 首先，我们定义起始节点。我们使用 `conversation`。
        // 这意味着这些是调用 `conversation` 节点后采用的边。
        "conversation",
        // 接下来，我们传递将确定 next 节点将调用哪个函数的函数。
        shouldContinue,
      )
      // 我们现在添加一条从 `summarize_conversation` 到 END 的正常边。
      // 这意味着在调用 `summarize_conversation` 后，我们将结束。
      .addEdge("summarize_conversation", END);

    // 最后，我们进行编译！
    const app = workflow.compile({ checkpointer: memory });
    ```
    :::

### 管理检查点

您可以查看和删除由 checkpointer 存储的信息。

#### 查看线程状态（检查点）

:::python
=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # 可选地为特定检查点提供 ID，
            # 否则将显示最新检查点
            # highlight-next-line
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

        }
    }
    # highlight-next-line
    graph.get_state(config)
    ```

    ```
    StateSnapshot(
        values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]}, next=(),
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        created_at='2025-05-05T16:01:24.680462+00:00',
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        tasks=(),
        interrupts=()
    )
    ```

=== "Checkpointer API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # 可选地为特定检查点提供 ID，
            # 否则将显示最新检查点
            # highlight-next-line
            # "checkpoint_id": "1f029ca3-1f5b-6704-8004-820c16b69a5a"

        }
    }
    # highlight-next-line
    checkpointer.get_tuple(config)
    ```

    ```
    CheckpointTuple(
        config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
        checkpoint={
            'v': 3,
            'ts': '2025-05-05T16:01:24.680462+00:00',
            'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
            'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
            'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today?), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
        },
        metadata={
            'source': 'loop',
            'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}},
            'step': 4,
            'parents': {},
            'thread_id': '1'
        },
        parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
        pending_writes=[]
    )
    ```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
    // 可选地为特定检查点提供 ID，
    // 否则将显示最新检查点
    // checkpoint_id: "1f029ca3-1f5b-6704-8004-820c16b69a5a"
  },
};
await graph.getState(config);
```

```
{
  values: { messages: [HumanMessage(...), AIMessage(...), HumanMessage(...), AIMessage(...)] },
  next: [],
  config: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1f5b-6704-8004-820c16b69a5a' } },
  metadata: {
    source: 'loop',
    writes: { call_model: { messages: AIMessage(...) } },
    step: 4,
    parents: {},
    thread_id: '1'
  },
  createdAt: '2025-05-05T16:01:24.680462+00:00',
  parentConfig: { configurable: { thread_id: '1', checkpoint_ns: '', checkpoint_id: '1f029ca3-1790-6b0a-8003-baf965b6a38f' } },
  tasks: [],
  interrupts: []
}
```

:::

#### 查看线程（检查点）的历史记录

:::python
=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1"
        }
    }
    # highlight-next-line
    list(graph.get_state_history(config))
    ```

    ```
    [
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            next=(),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:24.680462+00:00',
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")]},
            next=('call_model',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863421+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8ab4155e-6b15-b885-9ce5-bed69a2c305c', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Your name is Bob.')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=('__start__',),
            config={...},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.863173+00:00',
            parent_config={...}
            tasks=(PregelTask(id='24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "what's my name?"}]}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]},
            next=(),
            config={...},
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:23.862295+00:00',
            parent_config={...}
            tasks=(),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': [HumanMessage(content="hi! I'm bob")]},
            next=('call_model',),
            config={...},
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.278960+00:00',
            parent_config={...}
            tasks=(PregelTask(id='8cbd75e0-3720-b056-04f7-71ac805140a0', name='call_model', path=('__pregel_pull', 'call_model'), error=None, interrupts=(), state=None, result={'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}),),
            interrupts=()
        ),
        StateSnapshot(
            values={'messages': []},
            next=('__start__',),
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            created_at='2025-05-05T16:01:22.277497+00:00',
            parent_config=None,
            tasks=(PregelTask(id='d458367b-8265-812c-18e2-33001d199ce6', name='__start__', path=('__pregel_pull', '__start__'), error=None, interrupts=(), state=None, result={'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}),),
            interrupts=()
        )
    ]
    ```

=== "Checkpointer API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1"
        }
    }
    # highlight-next-line
    list(checkpointer.list(config))
    ```

    ```
    [
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1f5b-6704-8004-820c16b69a5a'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:24.680462+00:00',
                'id': '1f029ca3-1f5b-6704-8004-820c16b69a5a',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Your name is Bob.')}}, 'step': 4, 'parents': {}, 'thread_id': '1'},
            parent_config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-1790-6b0a-8003-baf965b6a38f'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863421+00:00',
                'id': '1f029ca3-1790-6b0a-8003-baf965b6a38f',
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 3, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8ab4155e-6b15-b885-9ce5-bed69a2c305c', 'messages', AIMessage(content='Your name is Bob.'))]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.863173+00:00',
                'id': '1f029ca3-1790-616e-8002-9e021694a0cd',
                'channel_versions': {'__start__': '00000000000000000000000000000004.0.5736472536395331', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}, 'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "what's my name?"}]}}, 'step': 2, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'messages', [{'role': 'user', 'content': "what's my name?"}]), ('24ba39d6-6db1-4c9b-f4c5-682aeaf38dcd', 'branch:to:call_model', None)]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:23.862295+00:00',
                'id': '1f029ca3-178d-6f54-8001-d7b180db0c89',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000003.0.7056767754077798', 'branch:to:call_model': '00000000000000000000000000000003.0.22059023329132854'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')]}
            },
            metadata={'source': 'loop', 'writes': {'call_model': {'messages': AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?')}}, 'step': 1, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[]
        ),
        CheckpointTuple(
            config={...},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.278960+00:00',
                'id': '1f029ca3-0874-6612-8000-339f2abc83b1',
                'channel_versions': {'__start__': '00000000000000000000000000000002.0.18673090920108737', 'messages': '00000000000000000000000000000002.0.30296526818059655', 'branch:to:call_model': '00000000000000000000000000000002.0.9300422176788571'},
                'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000001.0.7040775356287469'}},
                'channel_values': {'messages': [HumanMessage(content="hi! I'm bob")], 'branch:to:call_model': None}
            },
            metadata={'source': 'loop', 'writes': None, 'step': 0, 'parents': {}, 'thread_id': '1'},
            parent_config={...},
            pending_writes=[('8cbd75e0-3720-b056-04f7-71ac805140a0', 'messages', AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'))]
        ),
        CheckpointTuple(
            config={'configurable': {'thread_id': '1', 'checkpoint_ns': '', 'checkpoint_id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565'}},
            checkpoint={
                'v': 3,
                'ts': '2025-05-05T16:01:22.277497+00:00',
                'id': '1f029ca3-0870-6ce2-bfff-1f3f14c3e565',
                'channel_versions': {'__start__': '00000000000000000000000000000001.0.7040775356287469'},
                'versions_seen': {'__input__': {}},
                'channel_values': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}
            },
            metadata={'source': 'input', 'writes': {'__start__': {'messages': [{'role': 'user', 'content': "hi! I'm bob"}]}}, 'step': -1, 'parents': {}, 'thread_id': '1'},
            parent_config=None,
            pending_writes=[('d458367b-8265-812c-18e2-33001d199ce6', 'messages', [{'role': 'user', 'content': "hi! I'm bob"}]), ('d458367b-8265-812c-18e2-33001d199ce6', 'branch:to:call_model', None)]
        )
    ]
    ```

:::

:::js

```typescript
const config = {
  configurable: {
    thread_id: "1",
  },
};

const history = [];
for await (const state of graph.getStateHistory(config)) {
  history.push(state);
}
```

:::

#### 删除线程的所有检查点

:::python

```python
thread_id = "1"
checkpointer.delete_thread(thread_id)
```

:::

:::js

```typescript
const threadId = "1";
await checkpointer.deleteThread(threadId);
```

:::

:::python

## 预构建的记忆工具

**LangMem** 是 LangChain 维护的一个库，提供用于在代理中管理长期记忆的工具。有关用法示例，请参阅[LangMem 文档](https://langchain-ai.github.io/langmem/)。

:::