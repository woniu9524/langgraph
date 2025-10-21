# LangGraph Python SDK

该存储库包含用于与 LangSmith Deployment REST API 交互的 Python SDK。

## 快速入门

要开始使用 Python SDK，请[安装该软件包](https://pypi.org/project/langgraph-sdk/)

```bash
pip install -U langgraph-sdk
```

您需要一个正在运行的 LangGraph API 服务器。如果您使用 `langgraph-cli` 在本地运行服务器，SDK 将自动指向 `http://localhost:8123`，否则
在创建客户端时需要指定服务器 URL。

```python
from langgraph_sdk import get_client

# 如果您正在使用远程服务器，请使用 `get_client(url=REMOTE_URL)` 初始化客户端
client = get_client()

# 列出所有助手
assistants = await client.assistants.search()

# 我们会自动为配置文件中注册的每个图创建一个助手。
agent = assistants[0]

# 启动一个新线程
thread = await client.threads.create()

# 开始流式运行
input = {"messages": [{"role": "human", "content": "what's the weather in la"}]}
async for chunk in client.runs.stream(thread['thread_id'], agent['assistant_id'], input=input):
    print(chunk)
```