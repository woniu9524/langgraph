# 如何部署自托管控制平面

在部署之前，请查阅关于自托管控制平面部署选项的[概念指南](../../concepts/langgraph_self_hosted_control_plane.md)。

!!! info "重要提示"
    自托管控制平面部署选项需要[企业版](../../concepts/plans.md)套餐。

## 先决条件

1. 你正在使用 Kubernetes。
1. 你已部署自托管的 LangSmith。
1. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 在[本地测试你的应用程序](../../tutorials/langgraph-platform/local-server.md)。
1. 使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 构建 Docker 镜像（即 `langgraph build`），并将其推送到你的 Kubernetes 集群可以访问的注册表中。
1. 你的集群上已安装 `KEDA`。

         helm repo add kedacore https://kedacore.github.io/charts
         helm install keda kedacore/keda --namespace keda --create-namespace
1. Ingress 配置
    1. 你必须为你的 LangSmith 实例设置一个入口。所有代理都将作为 Kubernetes 服务部署在此入口之后。
    1. 你可以使用此指南为你的实例[设置入口](https://docs.smith.langchain.com/self_hosting/configuration/ingress)。
1. 你的集群中有足够的空间来容纳多个部署。建议使用 `Cluster-Autoscaler` 来自动配置新节点。
1. 你的集群中有可用的动态 PV provisioner 或 PV。你可以通过运行以下命令进行验证：

        kubectl get storageclass

## 设置

1. 作为配置自托管 LangSmith 实例的一部分，你需要启用 `langgraphPlatform` 选项。这将配置一些关键资源。
    1. `listener`：这是一个服务，它监听来自[控制平面](../../concepts/langgraph_control_plane.md)的部署更改，并创建/更新下游的 CRD。
    1. `LangGraphPlatform CRD`：LangGraph 平台部署的 CRD。它包含管理 LangGraph 平台部署实例的规范。
    1. `operator`：此 Operator 处理 LangGraph 平台 CRD 的更改。
    1. `host-backend`：这是[控制平面](../../concepts/langgraph_control_plane.md)。
1. 该图表还将使用两个附加镜像。使用最新发布版本中指定的镜像。

        hostBackendImage:
          repository: "docker.io/langchain/hosted-langserve-backend"
          pullPolicy: IfNotPresent
        operatorImage:
          repository: "docker.io/langchain/langgraph-operator"
          pullPolicy: IfNotPresent

1. 在你的 LangSmith 配置文件中（通常是 `langsmith_config.yaml`），启用 `langgraphPlatform` 选项。请注意，你还必须具有有效的入口设置：

        config:
          langgraphPlatform:
            enabled: true
            langgraphPlatformLicenseKey: "YOUR_LANGGRAPH_PLATFORM_LICENSE_KEY"
1. 在你的 `values.yaml` 文件中，配置 `hostBackendImage` 和 `operatorImage` 选项（如果你需要镜像镜像）。

1. 你还可以通过覆盖[此处](https://github.com/langchain-ai/helm/blob/main/charts/langsmith/values.yaml#L898)的基础模板来配置代理的基础模板。
1. 你从[控制平面 UI](../../concepts/langgraph_control_plane.md#control-plane-ui)创建部署。