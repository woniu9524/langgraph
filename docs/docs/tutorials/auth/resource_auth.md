# 让对话私密化

在本教程中，你将扩展[上一教程创建的聊天机器人](getting_started.md)，为每个用户提供他们自己的私密对话。你将添加[资源级访问控制](../../concepts/auth.md#single-owner-resources)，以便用户只能看到他们自己的对话。

![Authorization handlers](./img/authorization.png)

## 先决条件

在开始本教程之前，请确保你已成功运行[第一个教程中的机器人](getting_started.md)。

## 1. 添加资源授权

回想一下，在上一教程中，[`Auth`](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth) 对象允许你注册一个[身份验证函数](../../concepts/auth.md#authentication)，LangGraph Platform 使用该函数来验证传入请求中的 bearer token。现在，你将使用它来注册一个**授权**处理程序。

授权处理程序是在身份验证成功**之后**运行的函数。这些处理程序可以为资源添加[元数据](../../concepts/auth.md#filter-operations)（例如谁拥有它们）并过滤用户可以看到的内容。

更新你的 `src/security/auth.py` 文件，并添加一个授权处理程序以在每次请求时运行：

```python hl_lines="29-39" title="src/security/auth.py"
from langgraph_sdk import Auth

# 保留我们来自上一教程的测试用户
VALID_TOKENS = {
    "user1-token": {"id": "user1", "name": "Alice"},
    "user2-token": {"id": "user2", "name": "Bob"},
}

auth = Auth()


@auth.authenticate
async def get_current_user(authorization: str | None) -> Auth.types.MinimalUserDict:
    """我们上一教程中的身份验证处理程序。"""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"

    if token not in VALID_TOKENS:
        raise Auth.exceptions.HTTPException(status_code=401, detail="无效的 token")

    user_data = VALID_TOKENS[token]
    return {
        "identity": user_data["id"],
    }


@auth.on
async def add_owner(
    ctx: Auth.types.AuthContext,  # 包含当前用户信息
    value: dict,  # 正在创建/访问的资源
):
    """使资源与其创建者私密化。"""
    # 示例：
    # ctx: AuthContext(
    #     permissions=[],
    #     user=ProxyUser(
    #         identity='user1',
    #         is_authenticated=True,
    #         display_name='user1'
    #     ),
    #     resource='threads',
    #     action='create_run'
    # )
    # value: 
    # {
    #     'thread_id': UUID('1e1b2733-303f-4dcd-9620-02d370287d72'),
    #     'assistant_id': UUID('fe096781-5601-53d2-b2f6-0d3403f7e9ca'),
    #     'run_id': UUID('1efbe268-1627-66d4-aa8d-b956b0f02a41'),
    #     'status': 'pending',
    #     'metadata': {},
    #     'prevent_insert_if_inflight': True,
    #     'multitask_strategy': 'reject',
    #     'if_not_exists': 'reject',
    #     'after_seconds': 0,
    #     'kwargs': {
    #         'input': {'messages': [{'role': 'user', 'content': 'Hello!'}]},
    #         'command': None,
    #         'config': {
    #             'configurable': {
    #                 'langgraph_auth_user': ... 你的用户对象...
    #                 'langgraph_auth_user_id': 'user1'
    #             }
    #         },
    #         'stream_mode': ['values'],
    #         'interrupt_before': None,
    #         'interrupt_after': None,
    #         'webhook': None,
    #         'feedback_keys': None,
    #         'temporary': False,
    #         'subgraphs': False
    #     }
    # }

    # 执行 2 件事：
    # 1. 将用户 ID 添加到资源的元数据中。每个 LangGraph 资源都有一个 `metadata` 字典，该字典会随资源一起持久化。
    # 此元数据对于读写操作中的过滤非常有用
    # 2. 返回一个允许用户只查看自己资源的过滤器
    filters = {"owner": ctx.user.identity}
    metadata = value.setdefault("metadata", {})
    metadata.update(filters)

    # 只允许用户查看自己的资源
    return filters
```

该处理程序接收两个参数：

1. `ctx` ([AuthContext](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.AuthContext))：包含当前 `user`、用户的 `permissions`、`resource`（“threads”、“crons”、“assistants”）以及正在执行的 `action`（“create”、“read”、“update”、“delete”、“search”、“create_run”）信息。
2. `value` (`dict`)：正在创建或访问的数据。此字典的内容取决于正在访问的资源和操作。有关如何进行更严格范围的访问控制的信息，请参阅下面的[添加范围授权处理程序](#scoped-authorization)。

请注意，这个简单的处理程序执行两项操作：

1. 将用户 ID 添加到资源的元数据中。
2. 返回一个元数据过滤器，以便用户只能看到他们拥有的资源。

## 2. 测试私密对话

测试你的授权。如果设置正确，你应该会看到所有 ✅ 的消息。请确保你的开发服务器正在运行（运行 `langgraph dev`）：

```python
from langgraph_sdk import get_client

# 为两个用户创建客户端
alice = get_client(
    url="http://localhost:2024",
    headers={"Authorization": "Bearer user1-token"}
)

bob = get_client(
    url="http://localhost:2024",
    headers={"Authorization": "Bearer user2-token"}
)

# 爱丽丝创建一个助手
alice_assistant = await alice.assistants.create()
print(f"✅ Alice created assistant: {alice_assistant['assistant_id']}")

# 爱丽丝创建一个对话并聊天
alice_thread = await alice.threads.create()
print(f"✅ Alice created thread: {alice_thread['thread_id']}")

await alice.runs.create(
    thread_id=alice_thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hi, this is Alice's private chat"}]}
)

# 鲍勃尝试访问爱丽丝的对话
try:
    await bob.threads.get(alice_thread["thread_id"])
    print("❌ Bob shouldn't see Alice's thread!")
except Exception as e:
    print("✅ Bob correctly denied access:", e)

# 鲍勃创建自己的对话
bob_thread = await bob.threads.create()
await bob.runs.create(
    thread_id=bob_thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hi, this is Bob's private chat"}]}
)
print(f"✅ Bob created his own thread: {bob_thread['thread_id']}")

# 列出对话 - 每个用户只看到他们自己的
alice_threads = await alice.threads.search()
bob_threads = await bob.threads.search()
print(f"✅ Alice sees {len(alice_threads)} thread")
print(f"✅ Bob sees {len(bob_threads)} thread")
```

输出：

```bash
✅ Alice created assistant: fc50fb08-78da-45a9-93cc-1d3928a3fc37
✅ Alice created thread: 533179b7-05bc-4d48-b47a-a83cbdb5781d
✅ Bob correctly denied access: Client error '404 Not Found' for url 'http://localhost:2024/threads/533179b7-05bc-4d48-b47a-a83cbdb5781d'
For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404
✅ Bob created his own thread: 437c36ed-dd45-4a1e-b484-28ba6eca8819
✅ Alice sees 1 thread
✅ Bob sees 1 thread
```

这意味着：

1. 每个用户都可以创建自己的对话并进行聊天。
2. 用户无法看到彼此的对话。
3. 列出对话时只显示用户自己的对话。

## 3. 添加范围授权处理程序 {#scoped-authorization}

宽泛的 `@auth.on` 处理程序会匹配所有[授权事件](../../concepts/auth.md#supported-resources)。这很简洁，但这意味着 `value` 字典的内容没有得到很好的范围界定，并且相同的用户级访问控制应用于每个资源。如果你想更精细地控制，你也可以控制特定资源上的特定操作。

更新 `src/security/auth.py` 以添加特定资源类型的处理程序：

```python
# 保留我们之前的处理程序...

from langgraph_sdk import Auth

@auth.on.threads.create
async def on_thread_create(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.threads.create.value,
):
    """创建对话时添加所有者。
    
    此处理程序在创建新对话时运行，并执行两件事：
    1. 在正在创建的对话上设置元数据以跟踪所有权
    2. 返回一个确保只有创建者才能访问它的过滤器
    """
    # 示例值：
    #  {'thread_id': UUID('99b045bc-b90b-41a8-b882-dabc541cf740'), 'metadata': {}, 'if_exists': 'raise'}

    # 将所有者元数据添加到正在创建的对话中
    # 此元数据与对话一起存储并持久化
    metadata = value.setdefault("metadata", {})
    metadata["owner"] = ctx.user.identity
    
    
    # 返回过滤器以仅将访问限制在创建者
    return {"owner": ctx.user.identity}

@auth.on.threads.read
async def on_thread_read(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.threads.read.value,
):
    """只允许用户读取他们自己的对话。
    
    此处理程序在读取操作时运行。我们不需要设置
    元数据，因为对话已存在 - 我们只需要
    返回一个过滤器以确保用户只能看到他们自己的对话。
    """
    return {"owner": ctx.user.identity}

@auth.on.assistants
async def on_assistants(
    ctx: Auth.types.AuthContext,
    value: Auth.types.on.assistants.value,
):
    # 出于说明目的，我们将拒绝所有请求
    # 接触助手的资源
    # 示例值：
    # {
    #     'assistant_id': UUID('63ba56c3-b074-4212-96e2-cc333bbc4eb4'),
    #     'graph_id': 'agent',
    #     'config': {},
    #     'metadata': {},
    #     'name': 'Untitled'
    # }
    raise Auth.exceptions.HTTPException(
        status_code=403,
        detail="用户缺乏必要的权限。",
    )

# 假设你将信息组织在存储中，例如 (user_id, resource_type, resource_id)
@auth.on.store()
async def authorize_store(ctx: Auth.types.AuthContext, value: dict):
    # 每个存储项的“namespace”字段是一个元组，你可以将其视为项的目录。
    namespace: tuple = value["namespace"]
    assert namespace[0] == ctx.user.identity, "未授权"
```

请注意，你现在有了特定的处理程序，而不是一个全局处理程序：

1. 创建对话
2. 读取对话
3. 访问助手

前三个匹配每个资源上的特定**操作**（请参阅[资源操作](../../concepts/auth.md#resource-specific-handlers)），而最后一个 (`@auth.on.assistants`) 匹配助手资源上的**任何**操作。对于每个请求，LangGraph 将运行最匹配资源和正在访问的操作的最具体处理程序。这意味着以上四个处理程序将被运行，而不是广泛范围的 "`@auth.on`" 处理程序。

尝试向你的测试文件添加以下测试代码：

```python
# ... 与之前相同
# 尝试创建一个助手。这应该会失败
try:
    await alice.assistants.create("agent")
    print("❌ Alice shouldn't be able to create assistants!")
except Exception as e:
    print("✅ Alice correctly denied access:", e)

# 尝试搜索助手。这也会失败
try:
    await alice.assistants.search()
    print("❌ Alice shouldn't be able to search assistants!")
except Exception as e:
    print("✅ Alice correctly denied access to searching assistants:", e)

# 爱丽丝仍然可以创建对话
alice_thread = await alice.threads.create()
print(f"✅ Alice created thread: {alice_thread['thread_id']}")
```

输出：

```bash
✅ Alice created thread: dcea5cd8-eb70-4a01-a4b6-643b14e8f754
✅ Bob correctly denied access: Client error '404 Not Found' for url 'http://localhost:2024/threads/dcea5cd8-eb70-4a01-a4b6-643b14e8f754'
For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/404
✅ Bob created his own thread: 400f8d41-e946-429f-8f93-4fe395bc3eed
✅ Alice sees 1 thread
✅ Bob sees 1 thread
✅ Alice correctly denied access:
For more information check: https://developer.mozilla.org/en-US/docs/Web/HTTP/Status/500
✅ Alice correctly denied access to searching assistants:
```

恭喜！你已经构建了一个聊天机器人，其中每个用户都有自己的私密对话。虽然此系统使用简单的基于 token 的身份验证，但这些授权模式将在实现任何真实身份验证系统时生效。在下一教程中，你将使用 OAuth2 将测试用户替换为真实用户帐户。

## 下一步

既然你可以控制对资源的访问，你可能想：

1. 继续[连接身份验证提供商](add_auth_server.md)，以添加真实用户帐户。
2. 阅读更多关于[授权模式](../../concepts/auth.md#authorization)的信息。
3. 查看[API 参考](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth)，了解本教程中使用到的接口和方法的详细信息。