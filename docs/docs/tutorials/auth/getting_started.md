# 设置自定义身份验证

在本教程中，我们将构建一个只允许特定用户访问的聊天机器人。我们将从 LangGraph 模板开始，并逐步添加基于令牌的安全性。到最后，您将拥有一个工作正常的聊天机器人，该机器人会在允许访问之前检查有效的令牌。

这是我们的身份验证系列的第一部分：

1. 设置自定义身份验证（您在这里） - 控制谁可以访问您的机器人
2. [让对话私密化](resource_auth.md) - 让用户进行私密对话
3. [连接身份验证提供程序](add_auth_server.md) - 添加真实用户帐户并通过 OAuth2 进行验证以用于生产

本指南假设您熟悉以下基本概念：

*   [**身份验证与访问控制**](../../concepts/auth.md)
*   [**LangGraph 平台**](../../concepts/langgraph_platform.md)

!!! note

    自定义身份验证仅适用于 LangGraph Platform SaaS 部署或 Enterprise Self-Hosted 部署。

## 1. 创建您的应用

使用 LangGraph 入门模板创建新的聊天机器人：

```bash
pip install -U "langgraph-cli[inmem]"
langgraph new --template=new-langgraph-project-python custom-auth
cd custom-auth
```

该模板为我们提供了一个占位符 LangGraph 应用。通过安装本地依赖项并运行开发服务器来尝试一下：

```shell
pip install -e .
langgraph dev
```

服务器将启动并在您的浏览器中打开工作室：

```
> - 🚀 API: http://127.0.0.1:2024
> - 🎨 Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
> - 📚 API Docs: http://127.0.0.1:2024/docs
>
> 此内存服务器专为开发和测试而设计。
> 如需生产使用，请使用 LangGraph Platform。
```

如果您将其自我托管在公共互联网上，任何人都可以访问它！

![无身份验证](./img/no_auth.png)


## 2. 添加身份验证

现在您有了一个基础的 LangGraph 应用，可以为其添加身份验证。

!!! note

    在本教程中，您将从一个硬编码的令牌开始作为示例。您将在第三个教程中实现一个“生产就绪”的身份验证方案。

[`Auth`](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth) 对象允许您注册一个身份验证函数，LangGraph 平台将在每次请求时运行该函数。此函数接收每个请求并决定接受还是拒绝。

创建一个新文件 `src/security/auth.py`。您的代码将在此处检查用户是否被允许访问您的机器人：

```python hl_lines="10 15-16" title="src/security/auth.py"
from langgraph_sdk import Auth

# 这是我们的玩具用户数据库。请勿在生产环境中使用此方法
VALID_TOKENS = {
    "user1-token": {"id": "user1", "name": "Alice"},
    "user2-token": {"id": "user2", "name": "Bob"},
}

# "Auth" 对象是一个容器，LangGraph 将使用它来标记我们的身份验证函数
auth = Auth()


# "authenticate" 装饰器指示 LangGraph 将此函数作为中间件调用
# 针对每个请求。这将决定请求是否被允许
@auth.authenticate
async def get_current_user(authorization: str | None) -> Auth.types.MinimalUserDict:
    """检查用户的令牌是否有效。"""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"
    # 检查令牌是否有效
    if token not in VALID_TOKENS:
        raise Auth.exceptions.HTTPException(status_code=401, detail="无效令牌")

    # 如果有效，则返回用户信息
    user_data = VALID_TOKENS[token]
    return {
        "identity": user_data["id"],
    }
```

请注意，您的[身份验证](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate)处理程序做了两件重要的事情：

1.  检查请求的 [Authorization 标头](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) 中是否提供了有效的令牌。
2.  返回用户的[身份标识](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.MinimalUserDict)。

现在通过将以下内容添加到 [`langgraph.json`](../../cloud/reference/cli.md#configuration-file) 配置中来告诉 LangGraph 使用身份验证：

```json hl_lines="7-9" title="langgraph.json"
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "auth": {
    "path": "src/security/auth.py:auth"
  }
}
```

## 3. 测试您的机器人

再次启动服务器进行测试：

```bash
langgraph dev --no-browser
```

如果您没有添加 `--no-browser`，工作室 UI 将在浏览器中打开。您可能想知道，工作室如何仍然能够连接到我们的服务器？默认情况下，我们还允许从 LangGraph 工作室访问，即使在使用自定义身份验证时也是如此。这使得在工作室中开发和测试您的机器人更加容易。您可以通过在身份验证配置中设置 `disable_studio_auth: "true"` 来删除此替代身份验证选项：

```json
{
    "auth": {
        "path": "src/security/auth.py:auth",
        "disable_studio_auth": "true"
    }
}
```

## 4. 与您的机器人聊天

现在您应该只能通过在请求标头中提供有效令牌来访问机器人。然而，用户仍然能够访问彼此的资源，直到您在下一节教程中添加[资源授权处理程序](../../concepts/auth.md#resource-specific-handlers)。

![身份验证，无授权处理程序](./img/authentication.png)

在文件或笔记本中运行以下代码：

```python
from langgraph_sdk import get_client

# 尝试不带令牌访问（应该会失败）
client = get_client(url="http://localhost:2024")
try:
    thread = await client.threads.create()
    print("❌ 应该在没有令牌的情况下失败！")
except Exception as e:
    print("✅ 正确阻止了访问：", e)

# 尝试使用有效令牌
client = get_client(
    url="http://localhost:2024", headers={"Authorization": "Bearer user1-token"}
)

# 创建一个对话并进行聊天
thread = await client.threads.create()
print(f"✅ 以 Alice 的身份创建了对话：{thread['thread_id']}")

response = await client.runs.create(
    thread_id=thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hello!"}]},
)
print("✅ 机器人已响应：")
print(response)
```

您应该看到：

1.  没有有效令牌，我们无法访问机器人。
2.  使用有效令牌，我们可以创建对话并进行聊天。

恭喜！您已经构建了一个只允许“经过身份验证”的用户访问的聊天机器人。虽然此系统尚未实现生产就绪的安全方案，但我们已经学习了如何控制对机器人访问的基本机制。在下一教程中，我们将学习如何让每个用户拥有自己的私密对话。

## 后续步骤

现在您可以控制谁访问您的机器人了，您可能想：

1.  继续教程，前往[让对话私密化](resource_auth.md)，了解资源授权。
2.  阅读更多关于[身份验证概念](../../concepts/auth.md)的信息。
3.  查看[API 参考](../../cloud/reference/sdk/python_sdk_ref.md)以获取更多身份验证详细信息。