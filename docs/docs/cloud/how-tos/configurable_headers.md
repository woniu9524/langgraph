# 可配置的 Headers

LangGraph 允许在运行时进行配置，以动态修改 Agent 的行为和权限。当使用 [LangGraph Platform](../quick_start.md) 时，您可以在请求体 (`config`) 或特定的请求 Header 中传递此配置。这使得可以根据用户身份或其他请求数据进行调整。

为了保护隐私，请通过 `langgraph.json` 文件中的 `http.configurable_headers` 部分来控制哪些 Header 会被传递到运行时配置中。

以下是如何自定义包含和排除的 Headers：

```json
{
  "http": {
    "configurable_headers": {
      "include": ["x-user-id", "x-organization-id", "my-prefix-*"],
      "exclude": ["authorization", "x-api-key"]
    }
  }
}
```

`include` 和 `exclude` 列表接受精确的 Header 名称，或使用 `*` 进行模式匹配以匹配任意数量的字符。为了您的安全，不支持其他任何正则表达式模式。

## 在您的图中进行使用

您可以在图中使用 `config` 参数来访问包含的 Headers：

```python
def my_node(state, config):
  organization_id = config["configurable"].get("x-organization-id")
  ...
```

或者通过从上下文获取（这在工具或任何其他嵌套函数中非常有用）。

```python
from langgraph.config import get_config

def search_everything(query: str):
  organization_id = get_config()["configurable"].get("x-organization-id")
  ...
```

您甚至可以使用此功能动态编译图。

```python
# my_graph.py.
import contextlib

@contextlib.asynccontextmanager
async def generate_agent(config):
  organization_id = config["configurable"].get("x-organization-id")
  if organization_id == "org1":
    graph = ...
    yield graph
  else:
    graph = ...
    yield graph

```

```json
{
  "graphs": {"agent": "my_grph.py:generate_agent"}
}
```

### 退出可配置的 Headers

如果您想退出可配置的 Headers，只需在 `exclude` 列表中设置一个通配符模式：

```json
{
  "http": {
    "configurable_headers": {
      "exclude": ["*"]
    }
  }
}
```

这将排除所有 Headers 不被添加到您的运行配置中。

请注意，排除项优先于包含项。