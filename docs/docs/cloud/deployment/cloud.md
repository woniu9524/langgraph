# 如何部署到 Cloud SaaS

在部署之前，请回顾 [Cloud SaaS](../../concepts/langgraph_cloud.md) 部署选项的概念指南。

## 前提条件

1. LangGraph Platform 应用是从 GitHub 仓库部署的。将 LangGraph Platform 应用配置并上传到 GitHub 仓库，以便将其部署到 LangGraph Platform。
1. [确认 LangGraph API 在本地运行](../../tutorials/langgraph-platform/local-server.md)。如果 API 运行不成功（例如 `langgraph dev`），部署到 LangGraph Platform 也会失败。

## 创建新部署

从 [LangSmith UI](https://smith.langchain.com/) 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 在右上角，选择 `+ New Deployment` 来创建新部署。
1. 在 `Create New Deployment` 面板中，填写必填字段。
    1. `Deployment details`
        1. 选择 `Import from GitHub` 并按照 GitHub OAuth 工作流程操作，安装并授权 LangChain 的 `hosted-langserve` GitHub 应用来访问选定的仓库。安装完成后，返回 `Create New Deployment` 面板，从下拉菜单中选择要部署的 GitHub 仓库。**注意**: 安装 LangChain 的 `hosted-langserve` GitHub 应用的用户必须是组织或账户的 [所有者](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners)。
        1. 为部署指定一个名称。
        1. 指定所需的 `Git Branch`。部署与一个分支关联。创建新版本时，将部署链接分支的代码。之后可以在 [部署设置](#deployment-settings) 中更新分支。
        1. 指定 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 的完整路径，包括文件名。例如，如果 `langgraph.json` 文件位于仓库的根目录，只需指定 `langgraph.json`。
        1. 勾选/取消勾选 `Automatically update deployment on push to branch` 复选框。如果选中，当更改推送到指定 `Git Branch` 时，部署将自动更新。此设置之后可以在 [部署设置](#deployment-settings) 中启用/禁用。
    1. 选择所需的 `Deployment Type`。
        1. `Development` 部署用于非生产用例，资源最少。
        1. `Production` 部署可服务高达 500 个请求/秒，并提供高可用性存储和自动备份。
    1. 确定部署是否应该 `Shareable through LangGraph Studio`。
        1. 如果未选中，则只能通过工作区的有效 LangSmith API 密钥访问部署。
        1. 如果选中，则 LangGraph Studio 中的任何 LangSmith 用户都可以访问部署。将提供一个指向 LangGraph Studio 中该部署的直接 URL，以便与其他 LangSmith 用户共享。
    1. 指定 `Environment Variables` 和 secrets。请参阅 [环境变量参考](../reference/env_var.md) 来为部署配置其他变量。
        1. 像 API 密钥（例如 `OPENAI_API_KEY`）这样的敏感值应指定为 secrets。
        1. 也可以指定额外的非 secret 环境变量。
    1. 会自动创建一个与部署同名的 LangSmith `Tracing Project`。
1. 在右上角，选择 `Submit`。几秒钟后，将出现 `Deployment` 视图，新部署将排队等待配置。

## 创建新版本

在 [创建新部署](#create-new-deployment) 时，默认会创建一个新版本。后续版本可用于部署新的代码更改。

从 [LangSmith UI](https://smith.langchain.com/) 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 选择一个现有部署来创建新版本。
1. 在 `Deployment` 视图的右上角，选择 `+ New Revision`。
1. 在 `New Revision` 模态框中，填写必填字段。
    1. 指定 [LangGraph API 配置文件](../reference/cli.md#configuration-file) 的完整路径，包括文件名。例如，如果 `langgraph.json` 文件位于仓库的根目录，只需指定 `langgraph.json`。
    1. 确定部署是否应该 `Shareable through LangGraph Studio`。
        1. 如果未选中，则只能通过工作区的有效 LangSmith API 密钥访问部署。
        1. 如果选中，则 LangGraph Studio 中的任何 LangSmith 用户都可以访问部署。将提供一个指向 LangGraph Studio 中该部署的直接 URL，以便与其他 LangSmith 用户共享。
    1. 指定 `Environment Variables` 和 secrets。将预填充现有的 secrets 和环境变量。参阅 [环境变量参考](../reference/env_var.md) 来为新版本配置其他变量。
        1. 添加新的 secrets 或环境变量。
        1. 删除现有的 secrets 或环境变量。
        1. 更新现有 secrets 或环境变量的值。
1. 选择 `Submit`。几秒钟后，`New Revision` 模态框将关闭，新版本将排队等待部署。

## 查看构建和服务器日志

构建和服务器日志对每个版本都可用。

从 `LangGraph Platform` 视图开始...

1. 在 `Revisions` 表中选择所需版本。右侧会滑出一个面板，默认选中 `Build` 选项卡，其中显示了该版本的构建日志。
1. 在面板中，选择 `Server` 选项卡以查看该版本的服务器日志。服务器日志仅在版本部署后可用。
1. 在 `Server` 选项卡中，根据需要调整日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为 `Last 7 days`。

## 查看部署指标

从 [LangSmith UI](https://smith.langchain.com/) 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 选择一个现有部署进行监控。
1. 选择 `Monitoring` 选项卡以查看部署指标。请参阅 [所有可用指标](../../concepts/langgraph_control_plane.md#monitoring)。
1. 在 `Monitoring` 选项卡中，根据需要使用日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为 `Last 15 minutes`。

## 中断版本

中断版本将停止该版本的部署。

!!! warning "未定义行为"
    中断的版本行为是未定义的。这只有在你需要部署新版本而目前有一个版本“卡住”在进行中时才有用。将来，此功能可能会被移除。

从 `LangGraph Platform` 视图开始...

1. 在 `Revisions` 表中，选择所需版本行右侧的菜单图标（三个点）。
1. 从菜单中选择 `Interrupt`。
1. 会出现一个模态框。请仔细阅读确认消息。选择 `Interrupt revision`。

## 删除部署

从 [LangSmith UI](https://smith.langchain.com/) 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 选择所需部署行右侧的菜单图标（三个点），然后选择 `Delete`。
1. 会出现一个 `Confirmation` 模态框。选择 `Delete`。

## 部署设置

从 `LangGraph Platform` 视图开始...

1. 在右上角，选择齿轮图标（`Deployment Settings`）。
1. 将 `Git Branch` 更新为所需分支。
1. 勾选/取消勾选 `Automatically update deployment on push to branch` 复选框。
1. 分支的创建/删除以及标签的创建/删除事件不会触发更新。只有推送到现有分支的提交才会触发更新。
1. 对同一分支的快速连续推送会将后续更新排队。一旦一个构建完成，最新的提交将开始构建，而其他排队的构建将被跳过。

## 添加或移除 GitHub 仓库

安装并授权 LangChain 的 `hosted-langserve` GitHub 应用后，可以修改应用的仓库访问权限以添加新仓库或移除现有仓库。如果创建了新仓库，可能需要显式添加它。

1. 从 GitHub 个人资料，导航到 `Settings` > `Applications` > `hosted-langserve` > 单击 `Configure`。
1. 在 `Repository access` 下，选择 `All repositories` 或 `Only select repositories`。如果选择了 `Only select repositories`，则必须显式添加新仓库。
1. 单击 `Save`。
1. 在创建新部署时，下拉菜单中的 GitHub 仓库列表将更新以反映仓库访问权限的更改。

## 白名单 IP 地址

2025 年 1 月 6 日之后创建的 `LangGraph Platform` 部署的所有流量都将通过 NAT 网关。
此 NAT 网关将拥有几个静态 IP 地址，具体取决于你部署的区域。请参阅下表获取要白名单的 IP 地址列表：

| US             | EU              |
|----------------|-----------------|
| 35.197.29.146  | 34.90.213.236   |
| 34.145.102.123 | 34.13.244.114   |
| 34.169.45.153  | 34.32.180.189   |
| 34.82.222.17   | 34.34.69.108    |
| 35.227.171.135 | 34.32.145.240   |
| 34.169.88.30   | 34.90.157.44    |
| 34.19.93.202   | 34.141.242.180  |
| 34.19.34.50    | 34.32.141.108   |