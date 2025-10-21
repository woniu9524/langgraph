# 追踪 (Tracing)

追踪 (Traces) 是指您的应用程序从输入到输出所经历的一系列步骤。这些单独的步骤中的每一个都代表一次运行 (run)。您可以使用 [LangSmith](https://smith.langchain.com/) 来可视化这些执行步骤。要使用它，请为您的应用程序启用追踪： [enable tracing for your application](../how-tos/enable-tracing.md)。这将使您能够：

- [调试本地运行的应用程序](../cloud/how-tos/clone_traces_studio.md)。
- [评估应用程序性能](../agents/evals.md)。
- [监控应用程序](https://docs.smith.langchain.com/observability/how_to_guides/dashboards)。

要开始使用，请在 [LangSmith](https://smith.langchain.com/) 注册一个免费账户。

## 了解更多

- [在 LangSmith 中可视化运行图 (Graph runs in LangSmith)](../how-tos/run-id-langsmith.md)
- [LangSmith 可观测性快速入门 (LangSmith Observability quickstart)](https://docs.smith.langchain.com/observability)
- [使用 LangGraph 进行追踪 (Trace with LangGraph)](https://docs.smith.langchain.com/observability/how_to_guides/trace_with_langgraph)
- [追踪概念指南 (Tracing conceptual guide)](https://docs.smith.langchain.com/observability/concepts#traces)