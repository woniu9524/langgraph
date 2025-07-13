## 组件

LangGraph 平台由协同工作的组件组成，这些组件支持 LangGraph 应用程序的开发、部署、调试和监控：

- [LangGraph Server](./langgraph_server.md)：该服务器定义了一个有主见的 API 和架构，其中包含了部署agentic 应用程序的最佳实践，让您可以专注于构建agent 逻辑，而不是开发服务器基础设施。
- [LangGraph CLI](./langgraph_cli.md)：LangGraph CLI 是一个命令行界面，用于与本地 LangGraph 进行交互。
- [LangGraph Studio](./langgraph_studio.md)：LangGraph Studio 是一个专门的 IDE，可以连接到 LangGraph Server，从而在本地实现应用程序的可视化、交互和调试。
- [Python/JS SDK](./sdk.md)：Python/JS SDK 提供了一种以编程方式与已部署的 LangGraph 应用程序交互的方法。
- [远程图谱](../how-tos/use-remote-graph.md)：远程图谱允许您与任何已部署的 LangGraph 应用程序进行交互，就像它在本地运行一样。
- [LangGraph 控制平面](./langgraph_control_plane.md)：LangGraph 控制平面指的是用户创建和更新 LangGraph Server 的控制平面 UI，以及支持 UI 体验的控制平面 API。
- [LangGraph 数据平面](./langgraph_data_plane.md)：LangGraph 数据平面指的是 LangGraph Server、每个服务器对应的基础设施，以及持续轮询 LangGraph 控制平面更新的“监听器”应用程序。

![LangGraph components](img/lg_platform.png)