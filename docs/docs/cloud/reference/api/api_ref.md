# LangGraph 服务器 API 参考

LangGraph 服务器 API 参考可在每个部署的 `/docs` 端点（例如 `http://localhost:8124/docs`）找到。

点击 <a href="/langgraph/cloud/reference/api/api_ref.html" target="_blank">此处</a> 查看 API 参考。

## 认证

对于部署到 LangGraph Platform 的情况，需要进行身份验证。在每次向 LangGraph 服务器发送请求时，请传递 `X-Api-Key` 头。该头的值应设置为 LangGraph 服务器部署所在组织的有效 LangSmith API 密钥。

示例 `curl` 命令：
```shell
curl --request POST \
  --url http://localhost:8124/assistants/search \
  --header 'Content-Type: application/json' \
  --header 'X-Api-Key: LANGSMITH_API_KEY' \
  --data '{
  "metadata": {},
  "limit": 10,
  "offset": 0
}'
```