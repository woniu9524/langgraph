!!! info "先决条件"

    - [LangGraph Studio 概览](../../../concepts/langgraph_studio.md)

LangGraph Studio 支持连接两种类型的图：

- 部署在 [LangGraph 平台](../../../cloud/quick_start.md)上的图
- 通过 [LangGraph 服务器](../../../tutorials/langgraph-platform/local-server.md)在本地运行的图。

LangGraph Studio 可从 LangSmith UI 中的 LangGraph Platform Deployments（LangGraph 平台部署）选项卡访问。

## 已部署的应用

对于已在 LangGraph 平台[部署](../../quick_start.md)的应用，您可以将其 Studio 作为该部署的一部分进行访问。为此，请在 LangSmith UI 中的 LangGraph 平台中导航到该部署，然后点击“LangGraph Studio”按钮。

这将加载与您的实时部署连接的 Studio UI，允许您创建、读取和更新该部署中的[线程](../../../concepts/persistence.md#threads)、[助手](../../../concepts/assistants.md)和[内存](../../../concepts//memory.md)。

## 本地开发服务器

要使用 LangGraph Studio 测试您本地运行的应用，请确保您的应用已按照[本指南](https://langchain-ai.github.io/langgraph/cloud/deployment/setup/)进行设置。

!!! info "LangSmith 追踪"
    对于本地开发，如果您不希望将数据追踪到 LangSmith，请在应用的 `.env` 文件中设置 `LANGSMITH_TRACING=false`。禁用追踪后，没有任何数据会离开您的本地服务器。

接下来，安装 [LangGraph CLI](../../../concepts/langgraph_cli.md)：

```
pip install -U "langgraph-cli[inmem]"
```

然后运行：

```
langgraph dev
```

!!! warning "浏览器兼容性"
    Safari 会阻止对 Studio 的 `localhost` 连接。要解决此问题，请使用 `--tunnel` 参数运行上述命令，通过安全隧道访问 Studio。

这将首先在本地启动 LangGraph 服务器，以内存模式运行。服务器将以监视模式运行，监听代码更改并自动重启。阅读此[参考](https://langchain-ai.github.io/langgraph/cloud/reference/cli/#dev)以了解启动 API 服务器的所有选项。

如果成功，您将看到以下日志：

> Ready!
>
> - API: [http://localhost:2024](http://localhost:2024/)
>
> - Docs: http://localhost:2024/docs
>
> - LangGraph Studio Web UI: https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024

运行后，您将自动跳转到 LangGraph Studio。

对于已运行的服务器，可以通过以下任一方式访问 Studio：

1.  直接导航到以下 URL：`https://smith.langchain.com/studio/?baseUrl=http://127.0.0.1:2024`。
2.  在 LangSmith 中，导航到 LangGraph Platform Deployments 选项卡，点击“LangGraph Studio”按钮，输入 `http://127.0.0.1:2024`，然后点击“Connect”。

如果在不同的主机或端口上运行服务器，只需更新 `baseUrl` 以匹配即可。

### (可选) 连接调试器

如需进行带有断点和变量检查的分步调试：

```bash
# 安装 debugpy 包
pip install debugpy

# 启用调试模式启动服务器
langgraph dev --debug-port 5678
```

然后附加您首选的调试器：

=== "VS Code"

    将此配置添加到 `launch.json`：

    ```json
    {
        "name": "Attach to LangGraph",
        "type": "debugpy",
        "request": "attach",
        "connect": {
          "host": "0.0.0.0",
          "port": 5678
        }
    }
    ```

=== "PyCharm"

    1. 转到 Run → Edit Configurations
    2. 点击 + 号并选择 “Python Debug Server”
    3. 设置 IDE host name：`localhost`
    4. 设置 port：`5678`（或在上一步中选择的端口号）
    5. 点击“OK”并开始调试

## 故障排查

有关入门问题，请参阅此[故障排查指南](../../../troubleshooting/studio.md)。

## 后续步骤

请参阅以下指南，了解有关如何使用 Studio 的更多信息：

- [运行应用](../invoke_studio.md)
- [管理助手](./manage_assistants.md)
- [管理线程](../threads_studio.md)
- [迭代提示](../iterate_graph_studio.md)
- [调试 LangSmith 追踪](../clone_traces_studio.md)
- [添加节点到数据集](../datasets_studio.md)