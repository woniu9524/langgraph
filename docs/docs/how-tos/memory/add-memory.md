# 添加和管理内存

AI 应用程序需要[内存](../../concepts/memory.md)来在多次交互中共享上下文。在 LangGraph 中，您可以添加两种内存：

- 将[短期内存](#add-short-term-memory)作为代理[状态](../../concepts/low_level.md#state)的一部分，以启用多轮对话。
- 将[长期内存](#add-long-term-memory)添加到会话中以存储用户特定或应用程序级别的数据。

## 添加短期内存

**短期内存**（线程级别[持久化](../../concepts/persistence.md)）使代理能够跟踪多轮对话。要添加短期内存：

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

### 在生产环境中使用

在生产环境中，请使用由数据库支持的检查点：

```python
from langgraph.checkpoint.postgres import PostgresSaver

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresSaver.from_conn_string(DB_URI) as checkpointer:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(checkpointer=checkpointer)
```

??? example "示例：使用 [Postgres](https://pypi.org/project/langgraph-checkpoint-postgres/) 检查点"

    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        您需要调用 `checkpointer.setup()` 才能首次使用 Postgres 检查点。

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

??? example "示例：使用 [MongoDB](https://pypi.org/project/langgraph-checkpoint-mongodb/) 检查点"

    ```
    pip install -U pymongo langgraph langgraph-checkpoint-mongodb
    ```

    !!! note "设置"

        要使用 MongoDB 检查点，您需要一个 MongoDB 集群。如果您还没有集群，请遵循[此指南](https://www.mongodb.com/docs/guides/atlas/cluster/)进行创建。

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

??? example "示例：使用 [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) 检查点"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        您需要调用 `checkpointer.setup()` 才能首次使用 Redis 检查点。


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

### 在子图中进行使用

如果您的图包含[子图](../../concepts/subgraphs.md)，您只需在编译父图时提供检查点。LangGraph 将自动将检查点传播到子图。

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

def node_1(state: State):
    return {"foo": "hi! " + state["foo"]}

builder = StateGraph(State)
# highlight-next-line
builder.add_node("node_1", subgraph)
builder.add_edge(START, "node_1")

checkpointer = InMemorySaver()
# highlight-next-line
graph = builder.compile(checkpointer=checkpointer)
```

如果您希望子图拥有自己的内存，则可以通过 `with checkpointer=True` 来编译它。这在[多代理](../../concepts/multi_agent.md)系统中很有用，如果您希望代理能够跟踪其内部消息历史记录。

```python
subgraph_builder = StateGraph(...)
# highlight-next-line
subgraph = subgraph_builder.compile(checkpointer=True)
```

### 在工具中读取短期内存 { #read-short-term }

LangGraph 允许代理在工具中访问其短期内存（状态）。

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

有关更多信息，请参阅[上下文](../../agents/context.md)指南。

### 从工具中写入短期内存 { #write-short-term }

要修改执行期间代理的短期内存（状态），您可以直接从工具返回状态更新。这对于持久化中间结果或使信息可供后续工具或提示访问非常有用。

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
    """在找到用户信息后使用此功能向用户致意。"""
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

## 添加长期内存

使用长期内存来存储跨对话的用户特定或应用程序特定的数据。

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

### 在生产环境中使用

在生产环境中，请使用由数据库支持的存储：

```python
from langgraph.store.postgres import PostgresStore

DB_URI = "postgresql://postgres:postgres@localhost:5442/postgres?sslmode=disable"
# highlight-next-line
with PostgresStore.from_conn_string(DB_URI) as store:
    builder = StateGraph(...)
    # highlight-next-line
    graph = builder.compile(store=store)
```

??? example "示例：使用 [Postgres](https://pypi.org/project/langgraph-checkpoint-postgres/) 存储"

    ```
    pip install -U "psycopg[binary,pool]" langgraph langgraph-checkpoint-postgres
    ```

    !!! Setup
        您需要调用 `store.setup()` 才能首次使用 Postgres 存储。

    === "同步"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.postgres import PostgresSaver
        # highlight-next-line
        from langgraph.store.postgres import PostgresStore
        from langgraph.store.base import BaseStore

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

                # 如果用户要求模型记住，则存储新回忆
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

                # 如果用户要求模型记住，则存储新回忆
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
                stream_mode="values"
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
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

??? example "示例：使用 [Redis](https://pypi.org/project/langgraph-checkpoint-redis/) 存储"

    ```
    pip install -U langgraph langgraph-checkpoint-redis
    ```

    !!! Setup
        您需要调用 `store.setup()` 才能首次使用 Redis 存储。


    === "同步"

        ```python
        from langchain_core.runnables import RunnableConfig
        from langchain.chat_models import init_chat_model
        from langgraph.graph import StateGraph, MessagesState, START
        from langgraph.checkpoint.redis import RedisSaver
        # highlight-next-line
        from langgraph.store.redis import RedisStore
        from langgraph.store.base import BaseStore

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

                # 如果用户要求模型记住，则存储新回忆
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

                # 如果用户要求模型记住，则存储新回忆
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
                stream_mode="values"
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
                stream_mode="values"
            ):
                chunk["messages"][-1].pretty_print()
        ```

### 在工具中读取长期内存 { #read-long-term }

```python title="代理可使用的查找用户信息的工具"
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

1. `InMemoryStore` 是一个将数据存储在内存中的存储。在生产环境中，您通常会使用数据库或其他持久化存储。请查阅[存储文档](../../reference/store.md)以获取更多选项。如果您使用 **LangGraph Platform** 进行部署，平台将为您提供生产就绪的存储。
2. 在此示例中，我们使用 `put` 方法向存储写入一些示例数据。有关更多详细信息，请参阅[BaseStore.put][langgraph.store.base.BaseStore.put] API 参考。
3. 第一个参数是命名空间。它用于将相关数据分组在一起。在此示例中，我们使用 `users` 命名空间来分组用户数据。
4. 命名空间内的键。此示例使用用户 ID 作为键。
5. 我们要为给定用户存储的数据。
6. `get_store` 函数用于访问存储。您可以从代码中的任何位置（包括工具和提示）调用它。此函数返回创建代理时传递给代理的存储。
7. `get` 方法用于从存储中检索数据。第一个参数是命名空间，第二个参数是键。这将返回一个 `StoreValue` 对象，其中包含值和有关该值元数据。
8. `store` 被传递给代理。这使代理能够在运行工具时访问存储。您也可以使用 `get_store` 函数从代码中的任何位置访问存储。

### 从工具中写入长期内存 { #write-long-term }

```python title="更新用户信息的工具示例"
from typing_extensions import TypedDict

from langgraph.config import get_store
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

# 您可以直接访问存储以获取值
store.get(("users",), "user_123").value
```

1. `InMemoryStore` 是一个将数据存储在内存中的存储。在生产环境中，您通常会使用数据库或其他持久化存储。请查阅[存储文档](../../reference/store.md)以获取更多选项。如果您使用 **LangGraph Platform** 进行部署，平台将为您提供生产就绪的存储。
2. `UserInfo` 类是一个 `TypedDict`，它定义了用户信息的数据结构。LLM 将使用它根据架构格式化响应。
3. `save_user_info` 函数是一个允许代理更新用户信息的工具。这对于聊天应用程序非常有用，用户可以在其中更新其个人资料信息。
4. `get_store` 函数用于访问存储。您可以从代码中的任何位置（包括工具和提示）调用它。此函数返回创建代理时传递给代理的存储。
5. `put` 方法用于将数据存储在存储中。第一个参数是命名空间，第二个参数是键。这将用户信息的存储在存储中。
6. `user_id` 在配置中传递。它用于标识正在更新信息的哪个用户。

### 使用语义搜索

在图的内存存储中启用语义搜索，让图代理按语义相似性搜索存储中的项目。

```python
from langchain.embeddings import init_embeddings
from langgraph.store.memory import InMemoryStore

# 创建启用了语义搜索的存储
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

??? example "带有语义搜索的长期内存"

    ```python
    from typing import Optional

    from langchain.embeddings import init_embeddings
    from langchain.chat_models import init_chat_model
    from langgraph.store.base import BaseStore
    from langgraph.store.memory import InMemoryStore
    from langgraph.graph import START, MessagesState, StateGraph

    llm = init_chat_model("openai:gpt-4o-mini")

    # 创建启用了语义搜索的存储
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
        # 根据用户的最后一条消息进行搜索
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

请参阅[此指南](../../cloud/deployment/semantic_search.md)以了解如何将语义搜索与 LangGraph 内存存储结合使用。

## 管理短期内存

通过启用[短期内存](#add-short-term-memory)，长对话可能会超出 LLM 的上下文窗口。常见解决方案包括：

*   [截断消息](#trim-messages)：删除前 N 条或后 N 条消息（在调用 LLM 之前）
*   [从 LangGraph 状态中删除](../../concepts/low_level.md#reducers)消息以永久删除
*   [摘要消息](#summarize-messages)：汇总历史记录中的早期消息，并用摘要替换它们
*   [管理检查点](#manage-checkpoints)以存储和检索消息历史记录
*   自定义策略（例如，消息过滤等）

这使得代理能够在不超出 LLM 上下文窗口的情况下跟踪对话。

### 截断消息

大多数 LLM 都有一个最大支持的上下文窗口（以令牌为单位）。决定何时截断消息的一种方法是计算消息历史记录中的令牌数，并在每次接近该限制时进行截断。如果您使用的是 LangChain，您可以使用 `trim_messages` 实用程序并指定要从列表中保留的令牌数，以及用于处理边界的 `strategy`（例如，保留最后 `max_tokens`）。

=== "在代理中"

    要在代理中截断消息历史记录，请使用 [`pre_model_hook`][langgraph.prebuilt.chat_agent_executor.create_react_agent] 和 [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) 函数：

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

    # 此函数将在每次调用 LLM 的节点之前被调用
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

=== "在工作流中"

    要截断消息历史记录，请使用 [`trim_messages`](https://python.langchain.com/api_reference/core/messages/langchain_core.messages.utils.trim_messages.html) 函数：

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

??? example "完整示例：截断消息"

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

### 删除消息

您可以从图状态中删除消息以管理消息历史记录。当您想要删除特定消息或清除整个消息历史记录时，这很有用。

要从图状态中删除消息，您可以使用 `RemoveMessage`。为了使 `RemoveMessage` 起作用，您需要使用[reducers](../../concepts/low_level.md#reducers)的 [`add_messages`][langgraph.graph.message.add_messages] 键，例如 [`MessagesState`](../../concepts/low_level.md#messagesstate)。

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

!!! warning

    删除消息时，**请确保**生成的消息历史记录有效。检查您使用的 LLM 提供商的限制。例如：
    
    *   某些提供商期望消息历史记录以 `user` 消息开头
    *   大多数提供商要求包含工具调用的 `assistant` 消息后面跟着相应的 `tool` 结果消息。

??? example "完整示例：删除消息"

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

### 摘要消息

如上所示，截断或删除消息的问题是您可能会丢失消息队列中的信息。因此，一些应用程序受益于使用聊天模型汇总消息历史记录的更复杂的方法。

![](../../concepts/img/memory/summary.png)

=== "在代理中"

    要在代理中汇总消息历史记录，请使用 [`pre_model_hook`][langgraph.prebuilt.chat_agent_executor.create_react_agent] 和预先构建的 [`SummarizationNode`](https://langchain-ai.github.io/langmem/reference/short_term/#langmem.short_term.SummarizationNode) 抽象：

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
        # 注意：我们添加此键以跟踪之前的摘要信息
        # 以确保我们不会在每次 LLM 调用时都进行摘要
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

    1. `InMemorySaver` 是一个将代理状态存储在内存中的检查点。在生产环境中，您通常会使用数据库或其他持久化存储。请查阅[检查点文档](../../reference/checkpoints.md)以获取更多选项。如果您使用 **LangGraph Platform** 进行部署，平台将为您提供生产就绪的检查点。
    2. `context` 键已添加到代理的状态中。该键包含用于摘要节点的簿记信息。它用于跟踪上次摘要信息，并确保代理不会在每次 LLM 调用时都进行摘要，这可能效率低下。
    3. `checkpointer` 已传递给代理。这使代理能够跨调用持久化其状态。
    4. `pre_model_hook` 设置为 `SummarizationNode`。此节点将在将消息历史记录发送到 LLM 之前对其进行汇总。摘要节点将自动处理汇总过程并使用新的摘要更新代理的状态。如果您愿意，可以替换为自定义实现。请参阅[create_react_agent][langgraph.prebuilt.chat_agent_executor.create_react_agent] API 参考以获取更多详细信息。
    5. `state_schema` 设置为 `State` 类，它是包含附加 `context` 键的自定义状态。


=== "在工作流中"

    可以使用提示和编排逻辑来汇总消息历史记录。例如，在 LangGraph 中，您可以扩展 [`MessagesState`](../../concepts/low_level.md#working-with-messages-in-graph-state) 以包含 `summary` 键：

    ```python
    from langgraph.graph import MessagesState
    class State(MessagesState):
        summary: str
    ```

    然后，您可以生成聊天历史记录的摘要，使用任何现有摘要作为下一个摘要的上下文。当 `messages` 状态键中累积了一定数量的消息后，可以调用此 `summarize_conversation` 节点。

    ```python
    def summarize_conversation(state: State):

        # 首先，我们获取现有的任何摘要
        summary = state.get("summary", "")

        # 创建我们的摘要提示
        if summary:

            # 已存在摘要
            summary_message = (
                f"这是到目前为止的对话摘要：{summary}\n\n"
                "请考虑上述新消息来扩展摘要："
            )

        else:
            summary_message = "创建以上对话的摘要："

        # 将提示添加到我们的历史记录中
        messages = state["messages"] + [HumanMessage(content=summary_message)]
        response = model.invoke(messages)

        # 删除除最后两条消息之外的所有消息
        delete_messages = [RemoveMessage(id=m.id) for m in state["messages"][:-2]]
        return {"summary": response.content, "messages": delete_messages}
    ```

??? example "完整示例：摘要消息"

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

    1. 我们将把运行摘要保存在 `context` 字段中
    (由 `SummarizationNode` 预期)。
    2. 定义仅用于过滤输入到 `call_model` 节点的私有状态。
    3. 我们在这里传递私有输入状态，以隔离摘要节点返回的消息。

    ```
    ================================== Ai Message ==================================

    From our conversation, I can see that you introduced yourself as Bob. That's the name you shared with me when we began talking.

    Summary: In this conversation, I was introduced to Bob, who then asked me to write a poem about cats. I composed a poem titled "The Mystery of Cats" that captured cats' graceful movements, independent nature, and their special relationship with humans. Bob then requested a similar poem about dogs, so I wrote "The Joy of Dogs," which highlighted dogs' loyalty, enthusiasm, and loving companionship. Both poems were written in a similar style but emphasized the distinct characteristics that make each pet special.
    ```



### 管理检查点

您可以查看和删除检查点存储的信息。

#### 查看线程状态（检查点）

=== "Graph/Functional API"

    ```python
    config = {
        "configurable": {
            # highlight-next-line
            "thread_id": "1",
            # 可选地提供特定检查点的 ID，
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
            # 可选地提供特定检查点的 ID，
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
            'channel_values': {'messages': [HumanMessage(content="hi! I'm bob"), AIMessage(content='Hi Bob! How are you doing today? Is there anything I can help you with?'), HumanMessage(content="what's my name?"), AIMessage(content='Your name is Bob.')]},
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

#### 查看线程的检查点历史记录

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
            values={'messages': [HumanMessage(content="hi! I'm bob")]},
            next=(),
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
                'channel_versions': {'__start__': '00000000000000000000000000000005.0.5290678567601859', 'messages': '00000000000000000000000000000006.0.3205149138784782', 'branch:to:call_model': '00000000000000000000000000000006.0.14611156755133758'}, 'versions_seen': {'__input__': {}, '__start__': {'__start__': '00000000000000000000000000000004.0.5736472536395331'}, 'call_model': {'branch:to:call_model': '00000000000000000000000000000005.0.1410174088651449'}},
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

#### 删除线程的所有检查点

```python
thread_id = "1"
checkpointer.delete_thread(thread_id)
```

## 预构建内存工具

**LangMem** 是 LangChain 维护的一个库，它提供了在代理中管理长期内存的工具。请参阅[LangMem 文档](https://langchain-ai.github.io/langmem/)以获取使用示例。