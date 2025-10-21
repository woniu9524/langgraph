## 组件

LangGraph 平台包含一系列协同工作的组件，以支持 LangGraph 应用的开发、部署、调试和监控：

- [LangGraph Server](./langgraph_server.md): 该服务器定义了一个固定的 API 和架构，并融入了部署 agent 应用的最佳实践，使您能够专注于构建 agent 逻辑，而非开发服务器基础设施。
- [LangGraph CLI](./langgraph_cli.md): LangGraph CLI 是一个命令行界面，用于与本地 LangGraph 进行交互。
- [LangGraph Studio](./langgraph_studio.md): LangGraph Studio 是一个专门的 IDE，可以连接 LangGraph Server，从而在本地实现应用的 [可视化](./studio/visualization.md)、交互和调试。
- [Python/JS SDK](./sdk.md): Python/JS SDK 提供了一种以编程方式与已部署的 LangGraph 应用进行交互的方式。
- [Remote Graph](../how-tos/use-remote-graph.md): RemoteGraph 允许您与任何已部署的 LangGraph 应用进行交互，就像它在本地运行一样。
- [LangGraph control plane](./langgraph_control_plane.md): LangGraph Control Plane 指的是用户创建和更新 LangGraph Servers 的 Control Plane UI，以及支持该 UI 体验的 Control Plane APIs。
- [LangGraph data plane](./langgraph_data_plane.md): LangGraph Data Plane 指的是 LangGraph Servers、每个服务器对应的基础设施，以及不断轮询 LangGraph Control Plane 更新的“监听器”应用程序。

![LangGraph components](img/lg_platform.png)