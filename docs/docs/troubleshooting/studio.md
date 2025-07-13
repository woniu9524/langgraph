# LangGraph Studio 排错指南

## :fontawesome-brands-safari:{ .safari } Safari 连接问题

Safari 会阻止 `localhost` 上的纯 HTTP 流量。当您运行 `langgraph dev` 并使用 Studio 时，可能会看到“Failed to load assistants”的错误。

### 解决方案 1：使用 Cloudflare Tunnel

=== "Python"

    ```shell
    pip install -U langgraph-cli>=0.2.6
    langgraph dev --tunnel
    ```

=== "JS"

    ```shell
    # 需要 @langchain/langgraph-cli>=0.0.26
    npx @langchain/langgraph-cli dev --tunnel
    ```

该命令会输出一个类似如下格式的 URL：

```shell
https://smith.langchain.com/studio/?baseUrl=https://hamilton-praise-heart-costumes.trycloudflare.com
```

在 Safari 中使用此 URL 来加载 Studio。其中，`baseUrl` 参数指定了您的代理服务器端点。

### 解决方案 2：使用 Chromium 浏览器

Chrome 及其他 Chromium 浏览器允许在 `localhost` 上使用 HTTP。您可以直接运行 `langgraph dev`，无需额外配置。

## :fontawesome-brands-brave:{ .brave } Brave 连接问题

在启用 Brave Shields 时，Brave 会阻止 `localhost` 上的纯 HTTP 流量。当您运行 `langgraph dev` 并使用 Studio 时，可能会看到“Failed to load assistants”的错误。

### 解决方案 1：禁用 Brave Shields

通过点击 URL 栏中的 Brave 图标来禁用 LangSmith 的 Brave Shields。

![Brave Shields](./img/brave-shields.png)

### 解决方案 2：使用 Cloudflare Tunnel

=== "Python"

    ```shell
    pip install -U langgraph-cli>=0.2.6
    langgraph dev --tunnel
    ```

=== "JS"

    ```shell
    # 需要 @langchain/langgraph-cli>=0.0.26
    npx @langchain/langgraph-cli dev --tunnel
    ```

该命令会输出一个类似如下格式的 URL：

```shell
https://smith.langchain.com/studio/?baseUrl=https://hamilton-praise-heart-costumes.trycloudflare.com
```

在 Brave 中使用此 URL 来加载 Studio。其中，`baseUrl` 参数指定了您的代理服务器端点。

## Graph 边 (Edge) 问题

未定义的条件边可能会在您的图中显示意外的连接。这是因为，在没有正确定义的情况下，LangGraph Studio 会假设条件边可以连接到所有其他节点。要解决此问题，请使用以下方法之一显式定义路由路径：

### 解决方案 1：路径映射 (Path Map)

定义路由输出与目标节点之间的映射关系：

=== "Python"

    ```python
    graph.add_conditional_edges("node_a", routing_function, {True: "node_b", False: "node_c"})
    ```

=== "Javascript"

    ```ts
    graph.addConditionalEdges("node_a", routingFunction, { true: "node_b", false: "node_c" });
    ```

### 解决方案 2：路由类型定义 (Python)

使用 Python 的 `Literal` 类型指定可能的路由目标：

```python
def routing_function(state: GraphState) -> Literal["node_b","node_c"]:
    if state['some_condition'] == True:
        return "node_b"
    else:
        return "node_c"
```