# 设置自定义认证

在本教程中，我们将构建一个只允许特定用户访问的聊天机器人。我们将从 LangGraph 模板开始，然后逐步添加基于令牌的安全机制。完成后，您将拥有一个在允许访问之前会检查有效令牌的工作聊天机器人。

这是我们的认证系列的第一部分：

1. 设置自定义认证（您在此处） - 控制谁可以访问您的机器人
2. [让对话私密化](resource_auth.md) - 让用户拥有私密对话
3. [连接认证提供商](add_auth_server.md) - 添加真实的用户账户并使用 OAuth2 进行生产验证

本指南假定您已基本熟悉以下概念：

- [**认证与访问控制**](../../concepts/auth.md)
- [**LangGraph 平台**](../../concepts/langgraph_platform.md)

!!! note

    自定义认证仅适用于 LangGraph Platform SaaS 部署或 Enterprise Self-Hosted 部署。

## 1. 创建您的应用

使用 LangGraph 入门模板创建一个新的聊天机器人：

:::python

```bash
pip install -U "langchain-cli[inmem]"
langchain app new --template=new-langchain-app-python custom-auth
cd custom-auth
```

:::

:::js

```bash
npx @langchain/langgraph-cli new --template=new-langchain-app-typescript custom-auth
cd custom-auth
```

:::

该模板为我们提供了一个 LangGraph 应用的占位符。尝试安装本地依赖项并运行开发服务器来测试它：

:::python

```shell
pip install -e .
langchain dev
```

:::

:::js

```shell
npm install
npm run langchain dev
```

:::

服务器将启动并在您的浏览器中打开 Studio：

```
> - 🚀 API: http://127.0.0.1:2024
> - 🎨 Studio UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024
> - 📚 API Docs: http://127.0.0.1:2024/docs
>
> This in-memory server is designed for development and testing.
> For production use, please use LangSmith Deployment.
```

如果您将其自托管到公共互联网上，任何人都可以访问它！

![No auth](./img/no_auth.png)

## 2. 添加认证

现在您拥有了一个基础的 LangGraph 应用，可以为其添加认证功能。

!!! note

    在本教程的示例中，您将从一个硬编码的令牌开始。您将在第三个教程中实现一个“生产级别”的认证方案。

:::python
[`Auth`](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth) 对象允许您注册一个认证函数，LangGraph 平台将在每次请求时运行该函数。此函数会接收每个请求并决定是接受还是拒绝。

创建一个新文件 `src/security/auth.py`。您的代码将在此处检查用户是否被允许访问您的机器人：

```python hl_lines="10 15-16" title="src/security/auth.py"
from langgraph_sdk import Auth

# 这是我们的演示用户数据库。请勿在生产环境中使用此方法
VALID_TOKENS = {
    "user1-token": {"id": "user1", "name": "Alice"},
    "user2-token": {"id": "user2", "name": "Bob"},
}

# “Auth”对象是 LangGraph 将用于标记我们的认证函数的容器
auth = Auth()


# `authenticate` 装饰器告诉 LangGraph 将此函数作为中间件调用
# 针对每个请求。这将决定请求是否被允许
@auth.authenticate
async def get_current_user(authorization: str | None) -> Auth.types.MinimalUserDict:
    """检查用户的令牌是否有效。"""
    assert authorization
    scheme, token = authorization.split()
    assert scheme.lower() == "bearer"
    # 检查令牌是否有效
    if token not in VALID_TOKENS:
        raise Auth.exceptions.HTTPException(status_code=401, detail="Invalid token")

    # 如果有效，则返回用户信息
    user_data = VALID_TOKENS[token]
    return {
        "identity": user_data["id"],
    }
```

请注意，您的 [认证](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.Auth.authenticate) 处理程序执行了两项重要操作：

1. 检查请求的 [Authorization 标头](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) 中是否提供了有效的令牌
2. 返回用户的 [身份](../../cloud/reference/sdk/python_sdk_ref.md#langgraph_sdk.auth.types.MinimalUserDict)
   :::

:::js
[`Auth`](../../cloud/reference/sdk/js_sdk_ref.md#Auth) 对象允许您注册一个认证函数，LangGraph 平台将在每次请求时运行该函数。此函数会接收每个请求并决定是接受还是拒绝。

创建一个新文件 `src/security/auth.ts`。您的代码将在此处检查用户是否被允许访问您的机器人：

```typescript title="src/security/auth.ts"
import { Auth, HTTPException} from "@langchain/langgraph-sdk";

// 这是我们的演示用户数据库。请勿在生产环境中使用此方法
const VALID_TOKENS: Record<string, { id: string; name: string }> = {
  "user1-token": { id: "user1", name: "Alice" },
  "user2-token": { id: "user2", name: "Bob" },
};

// “Auth”对象是 LangGraph 将用于标记我们的认证函数的容器
const auth = new Auth()
  // `authenticate` 方法告诉 LangGraph 将此函数作为中间件调用
  // 针对每个请求。这将决定请求是否被允许
  .authenticate((request) => {
    // 我们之前的教程中的认证处理程序。
    const apiKey = request.headers.get("x-api-key");
    if (!apiKey || !isValidKey(apiKey)) {
      throw new HTTPException(401, "Invalid API key");
    }

    const [scheme, token] = apiKey.split(" ");
    if (scheme.toLowerCase() !== "bearer") {
      throw new Error("Bearer token required");
    }

    if (!VALID_TOKENS[token]) {
      throw new HTTPException(401, "Invalid token");
    }

    const userData = VALID_TOKENS[token];
    return {
      identity: userData.id,
    };
  });

export { auth };
```

请注意，您的 [认证](../../cloud/reference/sdk/js_sdk_ref.md#Auth) 处理程序执行了两项重要操作：

1. 检查请求的 [Authorization 标头](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Authorization) 中是否提供了有效的令牌
2. 返回用户的 [身份](../../cloud/reference/sdk/js_sdk_ref.md#Auth.types.MinimalUserDict)
   :::

现在，通过将以下内容添加到 [`langgraph.json`](../../cloud/reference/cli.md#configuration-file) 配置文件，告诉 LangGraph 使用认证：

:::python

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

:::

:::js

```json hl_lines="7-9" title="langgraph.json"
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.ts:graph"
  },
  "env": ".env",
  "auth": {
    "path": "src/security/auth.ts:auth"
  }
}
```

:::

## 3. 测试您的机器人

再次启动服务器以测试所有功能：

```bash
langchain dev --no-browser
```

如果您没有添加 `--no-browser` 参数，Studio UI 将在浏览器中打开。您可能会想，Studio 如何仍然能够连接到我们的服务器？默认情况下，我们也允许从 LangGraph Studio 访问，即使在使用自定义认证时也是如此。这使得在 Studio 中开发和测试您的机器人更加容易。您可以通过在认证配置中设置 `disable_studio_auth: "true"` 来移除此备用认证选项：

:::python

```json
{
  "auth": {
    "path": "src/security/auth.py:auth",
    "disable_studio_auth": "true"
  }
}
```

:::

:::js

```json
{
  "auth": {
    "path": "src/security/auth.ts:auth",
    "disable_studio_auth": "true"
  }
}
```

:::

## 4. 与您的机器人聊天

现在，您只有在请求标头中提供了有效令牌时才能访问机器人。然而，在您在下一部分教程中添加 [资源授权处理程序](../../concepts/auth.md#resource-specific-handlers) 之前，用户仍然可以访问彼此的资源。

![Authentication, no authorization handlers](./img/authentication.png)

:::python
在文件或 notebook 中运行以下代码：

```python
from langgraph_sdk import get_client

# 尝试不带令牌（应失败）
client = get_client(url="http://localhost:2024")
try:
    thread = await client.threads.create()
    print("❌ 未带令牌应失败！")
except Exception as e:
    print("✅ 已正确阻止访问：", e)

# 尝试使用有效令牌
client = get_client(
    url="http://localhost:2024", headers={"Authorization": "Bearer user1-token"}
)

# 创建一个线程并聊天
thread = await client.threads.create()
print(f"✅ 以 Alice 的身份创建了线程：{thread['thread_id']}")

response = await client.runs.create(
    thread_id=thread["thread_id"],
    assistant_id="agent",
    input={"messages": [{"role": "user", "content": "Hello!"}]},
)
print("✅ 机器人已响应：")
print(response)
```

:::

:::js
在 TypeScript 文件中运行以下代码：

```typescript
import { Client } from "@langchain/langgraph-sdk";

async function testAuth() {
  // 尝试不带令牌（应失败）
  const clientWithoutToken = new Client({ apiUrl: "http://localhost:2024" });
  try {
    const thread = await clientWithoutToken.threads.create();
    console.log("❌ 未带令牌应失败！");
  } catch (e) {
    console.log("✅ 已正确阻止访问：", e);
  }

  // 尝试使用有效令牌
  const client = new Client({
    apiUrl: "http://localhost:2024",
    headers: { Authorization: "Bearer user1-token" },
  });

  // 创建一个线程并聊天
  const thread = await client.threads.create();
  console.log(`✅ 以 Alice 的身份创建了线程：${thread.thread_id}`);

  const response = await client.runs.create(thread.thread_id, "agent", {
    input: { messages: [{ role: "user", content: "Hello!" }] },
  });
  console.log("✅ 机器人已响应：");
  console.log(response);
}

testAuth().catch(console.error);
```

:::

您应该看到：

1. 没有有效令牌，我们无法访问机器人
2. 使用有效令牌，我们可以创建线程并聊天

恭喜！您已经构建了一个只能让“已认证”用户访问的聊天机器人。虽然此系统（尚未）实现生产级别的安全方案，但我们已经学习了控制机器人访问的基本机制。在下一个教程中，我们将学习如何为每个用户提供他们自己的私密对话。

## 后续步骤

既然您已经可以控制谁访问您的机器人，您可能想：

1. 通过前往[让对话私密化](resource_auth.md)继续本教程，以了解资源授权。
2. 阅读更多关于[认证概念](../../concepts/auth.md)的内容。

:::python
3. 查看[API 参考](../../cloud/reference/sdk/python_sdk_ref.md)，了解更多认证详情。
:::

:::js
3. 查看[API 参考](../../cloud/reference/sdk/js_sdk_ref.md)，了解更多认证详情。
:::