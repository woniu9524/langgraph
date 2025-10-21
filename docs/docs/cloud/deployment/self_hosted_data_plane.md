# 如何部署自托管数据平面

部署前，请查看关于自托管数据平面（Self-Hosted Data Plane）部署选项的[概念指南](../../concepts/langgraph_self_hosted_data_plane.md)。

!!! info "重要提示"
    自托管数据平面部署选项需要 [Enterprise](../../concepts/plans.md) 计划。

## 先决条件

1. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 在[本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
1. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 构建 Docker 镜像（例如 `langgraph build`），并将其推送到您的 Kubernetes 集群或 Amazon ECS 集群可以访问的注册表中。

## Kubernetes

### 先决条件
1. 您的集群已安装 `KEDA`。

        helm repo add kedacore https://kedacore.github.io/charts
        helm install keda kedacore/keda --namespace keda --create-namespace

1. 您的集群已安装有效的 `Ingress` 控制器。
1. 您的集群有足够的可用空间用于多个部署。建议使用 `Cluster-Autoscaler` 来自动配置新节点。
1. 您需要为两个控制平面 URL 启用出口流量。监听器会轮询这些端点以获取部署信息：

        https://api.host.langchain.com
        https://api.smith.langchain.com

### 设置

1. 提供您的 LangSmith 组织 ID。我们将为您的组织启用自托管数据平面。
1. 我们会提供一个 [Helm chart](https://github.com/langchain-ai/helm/tree/main/charts/langgraph-dataplane)，您可以使用它来设置您的 Kubernetes 集群。此 chart 包含几个重要组件。
    1. `langgraph-listener`: 这是一个监听 LangChain [控制平面](../../concepts/langgraph_control_plane.md)中部署变更的服务，并创建/更新下游 CRD。
    1. `LangGraphPlatform CRD`: LangGraph 平台部署的一个 CRD（自定义资源定义）。它包含管理 LangGraph 平台部署实例的规范。
    1. `langgraph-platform-operator`: 此操作符处理 LangGraph Platform CRD 的变更。
1. 配置您的 `langgraph-dataplane-values.yaml` 文件。

        config:
          langsmithApiKey: "" # 您的工作区的 API 密钥
          langsmithWorkspaceId: "" # 工作区 ID
          hostBackendUrl: "https://api.host.langchain.com" # 仅在欧盟区域时覆盖此项
          smithBackendUrl: "https://api.smith.langchain.com" # 仅在欧盟区域时覆盖此项

1. 部署 `langgraph-dataplane` Helm chart。

        helm repo add langchain https://langchain-ai.github.io/helm/
        helm repo update
        helm upgrade -i langgraph-dataplane langchain/langgraph-dataplane --values langgraph-dataplane-values.yaml

1. 如果部署成功，您将在命名空间中看到两个正在启动的服务。

        NAME                                          READY   STATUS              RESTARTS   AGE
        langgraph-dataplane-listener-7fccd788-wn2dx   0/1     Running             0          9s
        langgraph-dataplane-redis-0                   0/1     ContainerCreating   0          9s

1. 从[控制平面 UI](../../concepts/langgraph_control_plane.md#control-plane-ui) 创建一个部署。

## Amazon ECS

即将推出！