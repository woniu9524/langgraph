# 订阅指标和运营元数据出口

> **重要：仅限自托管**
> 本节仅适用于未处于离线模式的客户，并假定您正在使用自托管的 LangGraph Platform 实例。
> 这不适用于 SaaS 或混合部署。

自托管的 LangGraph Platform 实例将所有信息存储在本地，并且永远不会将敏感信息发送到您的网络之外。目前，我们仅根据您订单中的授权跟踪平台使用情况以用于计费目的。为了更好地远程支持我们的客户，我们需要允许流量出口到 `https://beacon.langchain.com`。

未来，我们将引入支持诊断功能，以帮助我们确保 LangGraph Platform 在您的环境中以最佳水平运行。

> **警告**
> **这将需要允许流量从您的网络出口到 `https://beacon.langchain.com`。**
> **如果使用 API 密钥，您还需要允许流量出口到 `https://api.smith.langchain.com` 或 `https://eu.api.smith.langchain.com` 以进行 API 密钥验证。**

通常，我们发送到 Beacon 的数据可分为以下几类：

- **订阅指标**
  - 订阅指标用于确定 LangSmith 的访问级别和使用情况。这包括但不限于：
    - 已执行节点数
    - 已执行运行数
    - 许可证密钥验证
- **运营元数据**
  - 此元数据将包含并收集上述订阅指标，以协助进行远程支持，使 LangChain 团队能够更有效、更主动地诊断和排除性能问题。

## 示例负载

为最大限度地提高透明度，我们在此提供示例负载：

### 许可证验证（如果使用企业许可证）

**端点：**

`POST beacon.langchain.com/v1/beacon/verify`

**请求：**

```json
{
  "license": "<YOUR_LICENSE_KEY>"
}
```

**响应：**

```json
{
  "token": "Valid JWT" // 短期 JWT 令牌，用于避免重复的许可证检查
}
```

### API 密钥验证（如果使用 LangSmith API 密钥）

**端点：**
`POST api.smith.langchain.com/auth`

**请求：**

```json
"Headers": {
  X-Api-Key: <YOUR_API_KEY>
}
```

**响应：**

```json
{
  "org_config": {
    "org_id": "3a1c2b6f-4430-4b92-8a5b-79b8b567bbc1",
    ... // 其他组织详细信息
  }
}
```

### 使用情况报告

**端点：**

`POST beacon.langchain.com/v1/metadata/submit`

**请求：**

```json
{
  "license": "<YOUR_LICENSE_KEY>",
  "from_timestamp": "2025-01-06T09:00:00Z",
  "to_timestamp": "2025-01-06T10:00:00Z",
  "tags": {
    "langgraph.python.version": "0.1.0",
    "langgraph_api.version": "0.2.0",
    "langgraph.platform.revision": "abc123",
    "langgraph.platform.variant": "standard",
    "langgraph.platform.host": "host-1",
    "langgraph.platform.tenant_id": "3a1c2b6f-4430-4b92-8a5b-79b8b567bbc1",
    "langgraph.platform.project_id": "c5b5f53a-4716-4326-8967-d4f7f7799735",
    "langgraph.platform.plan": "enterprise",
    "user_app.uses_indexing": "true",
    "user_app.uses_custom_app": "false",
    "user_app.uses_custom_auth": "true",
    "user_app.uses_thread_ttl": "true",
    "user_app.uses_store_ttl": "false"
  },
  "measures": {
    "langgraph.platform.runs": 150,
    "langgraph.platform.nodes": 450
  },
  "logs": []
}
```

**响应：**

```json
"204 No Content"
```

## 我们的承诺

LangChain 不会在订阅指标或运营元数据中存储任何敏感信息。所收集的任何数据都不会与第三方共享。如果您对正在发送的数据有任何疑虑，请联系您的客户经理。