# 如何部署自托管数据平面

部署前，请查看自托管数据Plane的[概念指南](../../concepts/langgraph_self_hosted_data_plane.md)部署选项。

!!! info "重要"
    自托管数据Plane部署选项需要[企业版](../../concepts/plans.md)计划。

## 先决条件

1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)在[本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
1. 使用[LangGraph CLI](../../concepts/langgraph_cli.md)构建Docker镜像（例如`langgraph build`）并将其推送到您的Kubernetes集群或Amazon ECS集群可以访问的注册表中。

## Kubernetes

### 先决条件
1. `KEDA`已安装在您的集群中。

        helm repo add kedacore https://kedacore.github.io/charts
        helm install keda kedacore/keda --namespace keda --create-namespace

1. 有一个有效的`Ingress`控制器已安装在您的集群中。
1. 您的集群中有足够的可用空间用于多个部署。建议使用`Cluster-Autoscaler`自动配置新节点。
1. 您需要启用到两个控制平面 URL 的出口流量。监听器会轮询这些端点以获取部署信息：

        https://api.host.langchain.com
        https://api.smith.langchain.com

### 设置

1. 提供您的LangSmith组织 ID。我们将为您的组织启用自托管数据Plane。
1. 我们将提供一个[Helm图表](https://github.com/langchain-ai/helm/tree/main/charts/langgraph-dataplane)，您可以使用它来设置您的Kubernetes集群。此图表包含几个重要组件：
    1. `langgraph-listener`：这是一个服务，用于监听LangChain的[控制平面](../../concepts/langgraph_control_plane.md)有关您部署的更改信息，并创建/更新下游的自定义资源（CRDs）。
    1. `LangGraphPlatform CRD`：LangGraph Platform部署的CRD。它包含了管理LangGraph Platform部署实例的规范。
    1. `langgraph-platform-operator`：此运算符处理LangGraph Platform CRD的更改。
1. 配置您的`langgraph-dataplane-values.yaml`文件。

        config:
          langgraphPlatformLicenseKey: "" # 您的LangGraph Platform许可证密钥
          langsmithApiKey: "" # 您的工作区的API密钥
          langsmithWorkspaceId: "" # 工作区ID
          hostBackendUrl: "https://api.host.langchain.com" # 仅在您使用欧盟区域时覆盖此项
          smithBackendUrl: "https://api.smith.langchain.com" # 仅在您使用欧盟区域时覆盖此项

1. 部署`langgraph-dataplane` Helm图表。

        helm repo add langchain https://langchain-ai.github.io/helm/
        helm repo update
        helm upgrade -i langgraph-dataplane langchain/langgraph-dataplane --values langgraph-dataplane-values.yaml

1. 如果成功，您将在您的命名空间中看到两个服务启动。

        NAME                                          READY   STATUS              RESTARTS   AGE
        langgraph-dataplane-listener-7fccd788-wn2dx   0/1     Running             0          9s
        langgraph-dataplane-redis-0                   0/1     ContainerCreating   0          9s

1. 您可以从[控制平面UI](../../concepts/langgraph_control_plane.md#control-plane-ui)创建部署。

## Amazon ECS

敬请期待！