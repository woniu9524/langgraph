# 自托管控制平面部署指南

部署前，请先阅读 [自托管控制平面部署选项的概念指南](../../concepts/langgraph_self_hosted_control_plane.md)。

!!! info "重要提示"
    自托管控制平面部署选项需要 [企业版](../../concepts/plans.md) 计划。

## 先决条件

1.  您正在使用 Kubernetes。
2.  您已部署自托管的 LangSmith。
3.  使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 在[本地测试您的应用程序](../../tutorials/langgraph-platform/local-server.md)。
4.  使用 [LangGraph CLI](../../concepts/langgraph_cli.md) 构建 Docker 镜像（例如 `langgraph build`）并将其推送到您的 Kubernetes 集群可以访问的注册表中。
5.  您的集群已安装 `KEDA`。

         helm repo add kedacore https://kedacore.github.io/charts
         helm install keda kedacore/keda --namespace keda --create-namespace
6.  Ingress 配置
    1.  您必须为您的 LangSmith 实例设置一个入口（ingress）。所有代理都将作为 Kubernetes 服务部署在此入口之后。
    1.  您可以使用此指南为您的实例 [设置入口](https://docs.smith.langchain.com/self_hosting/configuration/ingress)。
7.  您的集群中有足够的空间用于多个部署。建议使用 `Cluster-Autoscaler` 来自动配置新节点。
8.  您的集群中有可用的动态 PV（持久卷）提供程序或 PV。您可以通过运行以下命令进行验证：

        kubectl get storageclass

9.  您的网络需要能够访问 `https://beacon.langchain.com`。如果未运行在隔离模式（air-gapped mode）下，这是进行许可证验证和使用情况报告所必需的。有关详细信息，请参阅 [Egress 文档](../../cloud/deployment/egress.md)。

## 设置

1.  在配置自托管 LangSmith 实例时，启用 `langgraphPlatform` 选项。这将配置一些关键资源。
    1.  `listener`：这是一个监听[控制平面](../../concepts/langgraph_control_plane.md)对您的部署进行更改的服务，并创建/更新下游 CRD。
    1.  `LangGraphPlatform CRD`：LangGraph Platform 部署的 CRD。它包含管理 LangGraph Platform 部署实例的规范。
    1.  `operator`：此操作员处理 LangGraph Platform CRD 的更改。
    1.  `host-backend`：这是[控制平面](../../concepts/langgraph_control_plane.md)。
2.  图表还将使用另外两个镜像。请使用最新版本中指定的镜像。

        hostBackendImage:
          repository: "docker.io/langchain/hosted-langserve-backend"
          pullPolicy: IfNotPresent
        operatorImage:
          repository: "docker.io/langchain/langgraph-operator"
          pullPolicy: IfNotPresent

3.  在您的 LangSmith 配置文件（通常是 `langsmith_config.yaml`）中，启用 `langgraphPlatform` 选项。请注意，您还必须设置一个有效的入口：

        config:
          langgraphPlatform:
            enabled: true
            langgraphPlatformLicenseKey: "YOUR_LANGGRAPH_PLATFORM_LICENSE_KEY"
4.  在您的 `values.yaml` 文件中，配置 `hostBackendImage` 和 `operatorImage` 选项（如果您需要镜像）。

5.  您还可以通过此处 [覆盖基础模板](https://github.com/langchain-ai/helm/blob/main/charts/langsmith/values.yaml#L898)来配置代理的基础模板。
6.  您将从[控制平面 UI](../../concepts/langgraph_control_plane.md#control-plane-ui)创建部署。