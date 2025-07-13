# 添加自定义中间件

在将代理部署到 LangGraph 平台时，您可以为服务器添加自定义中间件，以处理诸如记录请求指标、注入或检查标头以及强制执行安全策略等问题，而无需修改核心服务器逻辑。这与[添加自定义路由](./custom_routes.md)的工作方式相同。您只需提供自己的 [`Starlette`](https://www.starlette.io/applications/) 应用程序（包括 [`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/) 和其他兼容的应用程序）。

添加中间件可让您全局拦截和修改部署中的请求和响应，无论是命中自定义端点还是内置的 LangGraph 平台 API。

下面是一个使用 FastAPI 的示例。

???+ note "仅支持 Python"

    我们目前仅在部署 Python 时支持自定义中间件，且需要 `langgraph-api>=0.0.26`。

## 创建应用程序

从**现有**的 LangGraph 平台应用程序开始，将以下中间件代码添加到您的 `webapp.py` 文件中。如果您是从头开始，可以使用 CLI 从模板创建新应用程序。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

拥有 LangGraph 项目后，请添加以下应用程序代码：

```python
# ./src/agent/webapp.py
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware

# highlight-next-line
app = FastAPI()

class CustomHeaderMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        response = await call_next(request)
        response.headers['X-Custom-Header'] = 'Hello from middleware!'
        return response

# 将中间件添加到应用程序
app.add_middleware(CustomHeaderMiddleware)
```

## 配置 `langgraph.json`

将以下内容添加到您的 `langgraph.json` 配置文件中。请确保路径指向您上面创建的 `webapp.py` 文件。

```json
{
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent/graph.py:graph"
  },
  "env": ".env",
  "http": {
    "app": "./src/agent/webapp.py:app"
  }
  // 其他配置选项，如 auth、store 等。
}
```

## 启动服务器

在本地测试服务器：

```bash
langgraph dev --no-browser
```

现在，您服务器的任何请求都将在其响应中包含自定义标头 `X-Custom-Header`。

## 部署

您可以按原样将此应用程序部署到 LangGraph 平台或您自托管的平台。

## 后续步骤

现在您已为部署添加了自定义中间件，您可以使用类似的技术添加[自定义路由](./custom_routes.md)或定义[自定义生命周期事件](./custom_lifespan.md)，以进一步自定义服务器的行为。