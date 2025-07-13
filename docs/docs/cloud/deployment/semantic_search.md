# 为您的 LangGraph 部署添加语义搜索

本指南将介绍如何为您的 LangGraph 部署的跨线程 [store](../../concepts/persistence.md#memory-store) 添加语义搜索，以便您的代理可以通过语义相似性搜索记忆和其他文档。

## 前提条件

- LangGraph 部署（请参阅 [如何部署](setup_pyproject.md)）
- 您的 embedding 提供商的 API 密钥（本例中为 OpenAI）
- `langchain >= 0.3.8`（如果您使用下面指定的字符串格式）

## 步骤

1. 更新您的 `langgraph.json` 配置文件，以包含 store 配置：

```json
{
    ...
    "store": {
        "index": {
            "embed": "openai:text-embedding-3-small",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

此配置：

- 使用 OpenAI 的 text-embedding-3-small 模型生成 embeddings
- 将 embedding 维度设置为 1536（与模型的输出匹配）
- 索引您存储数据中的所有字段（`["$"]` 表示索引所有字段，或指定特定字段，如 `["text", "metadata.title"]`）

2. 要使用上面的字符串 embedding 格式，请确保您的依赖项包含 `langchain >= 0.3.8`：

```toml
# 在 pyproject.toml 中
[project]
dependencies = [
    "langchain>=0.3.8"
]
```

或者如果使用 requirements.txt：

```
langchain>=0.3.8
```

## 使用方法

配置完成后，您可以在 LangGraph 节点中使用语义搜索。store 需要一个命名空间元组来组织记忆：

```python
def search_memory(state: State, *, store: BaseStore):
    # 使用语义相似性搜索 store
    # 命名空间元组有助于组织不同类型的记忆
    # 例如，("user_facts", "preferences") 或 ("conversation", "summaries")
    results = store.search(
        namespace=("memory", "facts"),  # 按类型组织记忆
        query="您的搜索查询",
        limit=3  # 要返回的结果数
    )
    return results
```

## 自定义 Embeddings

如果您想使用自定义 embeddings，您可以传递一个自定义 embedding 函数的路径：

```json
{
    ...
    "store": {
        "index": {
            "embed": "path/to/embedding_function.py:embed",
            "dims": 1536,
            "fields": ["$"]
        }
    }
}
```

部署将在此指定路径中查找函数。该函数必须是异步的，并接受字符串列表：

```python
# path/to/embedding_function.py
from openai import AsyncOpenAI

client = AsyncOpenAI()

async def aembed_texts(texts: list[str]) -> list[list[float]]:
    """自定义 embedding 函数必须：
    1. 是异步的
    2. 接受字符串列表
    3. 返回 float 数组列表（embeddings）
    """
    response = await client.embeddings.create(
        model="text-embedding-3-small",
        input=texts
    )
    return [e.embedding for e in response.data]
```

## 通过 API 查询

您也可以使用 LangGraph SDK 查询 store。由于 SDK 使用异步操作：

```python
from langgraph_sdk import get_client

async def search_store():
    client = get_client()
    results = await client.store.search_items(
        ("memory", "facts"),
        query="您的搜索查询",
        limit=3  # 要返回的结果数
    )
    return results

# 在异步上下文中调用
results = await search_store()
```