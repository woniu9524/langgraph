# 添加自定义路由

在将代理部署到 LangGraph 平台时，您的服务器会自动公开用于创建运行（runs）和线程（threads）、与长期记忆存储（long-term memory store）交互、管理可配置助手（configurable assistants）以及其他核心功能的路由（[查看所有默认 API 端点](../../cloud/reference/api/api_ref.md)）。

您可以通过提供自己的 [`Starlette`](https://www.starlette.io/applications/) 应用（包括 [`FastAPI`](https://fastapi.tiangolo.com/)、[`FastHTML`](https://fastht.ml/) 和其他兼容的应用）来添加自定义路由。您可以通过 `langgraph.json` 配置文件中应用的路径来让 LangGraph 平台识别它。

定义自定义应用对象后，您可以添加任何所需的路由，从而实现从 `/login` 端点的添加，到部署整个全栈 Web 应用，所有这些都可以在单个 LangGraph 服务器中完成。

下面是一个使用 FastAPI 的示例。

## 创建应用

从一个**现有**的 LangGraph 平台应用开始，将以下自定义路由代码添加到您的 Web 应用文件中。如果您是从头开始，可以使用 CLI 从模板创建一个新应用。

```bash
langgraph new --template=new-langgraph-project-python my_new_project
```

拥有 LangGraph 项目后，添加以下应用代码：

```python
# ./src/agent/webapp.py
from fastapi import FastAPI

# highlight-next-line
app = FastAPI()


@app.get("/hello")
def read_root():
    return {"Hello": "World"}

```

## 配置 `langgraph.json`

将以下内容添加到您的 `langgraph.json` 配置文件中。确保路径指向您上面创建的 `webapp.py` 文件中的 FastAPI 应用实例 `app`。

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

如果您在浏览器中导航到 `localhost:2024/hello`（`2024` 是默认的开发端口），您应该会看到 `/hello` 端点返回 `{"Hello": "World"}`。

!!! note "覆盖默认端点"

    您在应用中创建的路由具有优先于系统默认路由的优先级，这意味着您可以覆盖并重新定义任何默认端点的行为。

## 部署

您可以按原样将此应用部署到 LangGraph 平台或您自托管的平台。

## 后续步骤

现在您已经为部署添加了自定义路由，可以使用相同的技术进一步自定义服务器的行为，例如定义自定义 [中间件](./custom_middleware.md) 和 [生命周期事件](./custom_lifespan.md)。