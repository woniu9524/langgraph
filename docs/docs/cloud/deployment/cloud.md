# 如何部署到 Cloud SaaS

部署前，请回顾 Cloud SaaS 部署选项的概念指南。

## 先决条件

1. LangGraph Platform 应用程序是从 GitHub 仓库部署的。请配置 LangGraph Platform 应用程序并将其上传到 GitHub 仓库，以便部署到 LangGraph Platform。
1. [验证 LangGraph API 是否在本地运行](../../tutorials/langgraph-platform/local-server.md)。如果 API 未成功运行（即 `langgraph dev`），则部署到 LangGraph Platform 也会失败。

## 创建新部署

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 在右上角，选择 `+ New Deployment` 来创建新部署。
1. 在 `Create New Deployment` 面板中，填写必填字段。
    1. `Deployment details`
        1. 选择 `Import from GitHub`，然后按照 GitHub OAuth 工作流程安装并授权 LangChain 的 `hosted-langserve` GitHub 应用来访问选定的仓库。安装完成后，返回 `Create New Deployment` 面板，并从下拉菜单中选择要部署的 GitHub 仓库。**注意**：安装 LangChain 的 `hosted-langserve` GitHub 应用的 GitHub 用户必须是组织或账户的[所有者](https://docs.github.com/en/organizations/managing-peoples-access-to-your-organization-with-roles/roles-in-an-organization#organization-owners)。
        1. 指定部署的名称。
        1. 指定所需的 `Git Branch`。部署链接到一个分支。当创建新修订版本时，将部署链接分支的代码。之后可以在[部署设置](#deployment-settings)中更新分支。
        1. 指定包含文件名的 [LangGraph API 配置文件](../reference/cli.md#configuration-file)的完整路径。例如，如果文件 `langgraph.json` 在仓库的根目录下，只需指定 `langgraph.json`。
        1. 勾选/取消勾选复选框以 `Automatically update deployment on push to branch`。如果勾选，当更改推送到指定 `Git Branch` 时，部署将自动更新。此设置稍后可以在[部署设置](#deployment-settings)中启用/禁用。
    1. 选择所需的 `Deployment Type`。
        1. `Development` 部署适用于非生产用例，并配置了最少的资源。
        1. `Production` 部署可以处理高达 500 次/秒的请求，并配置了高可用性存储和自动备份。
    1. 确定部署是否应 `Shareable through LangGraph Studio`。
        1. 如果未勾选，则只能使用工作区的有效 LangSmith API 密钥访问部署。
        1. 如果勾选，任何 LangSmith 用户都可以通过 LangGraph Studio 访问部署。将提供一个直接指向 LangGraph Studio 的部署 URL，以便与其他人共享。
    1. 指定 `Environment Variables` 和 secrets。请参阅[环境变量参考](../reference/env_var.md)来配置部署的其他变量。
        1. 像 API 密钥（例如 `OPENAI_API_KEY`）这样的敏感值应指定为 secrets。
        1. 也可以指定额外的非 secret 环境变量。
    1. 将自动创建一个与部署同名的新的 LangSmith `Tracing Project`。
1. 在右上角，选择 `Submit`。几秒钟后，将显示 `Deployment` 视图，并且新部署将被排队进行配置。

## 创建新修订

当[创建新部署](#create-new-deployment)时，默认会创建一个新修订版本。后续可以创建修订版本来部署新的代码更改。

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 选择一个现有部署以创建新修订版本。
1. 在 `Deployment` 视图的右上角，选择 `+ New Revision`。
1. 在 `New Revision` 模态中，填写必填字段。
    1. 指定包含文件名的 [LangGraph API 配置文件](../reference/cli.md#configuration-file)的完整路径。例如，如果文件 `langgraph.json` 在仓库的根目录下，只需指定 `langgraph.json`。
    1. 确定部署是否应 `Shareable through LangGraph Studio`。
        1. 如果未勾选，则只能使用工作区的有效 LangSmith API 密钥访问部署。
        1. 如果勾选，任何 LangSmith 用户都可以通过 LangGraph Studio 访问部署。将提供一个直接指向 LangGraph Studio 的部署 URL，以便与其他人共享。
    1. 指定 `Environment Variables` 和 secrets。将预填充现有的 secrets 和环境变量。请参阅[环境变量参考](../reference/env_var.md)来配置修订版本的其他变量。
        1. 添加新的 secrets 或环境变量。
        1. 删除现有的 secrets 或环境变量。
        1. 更新现有 secrets 或环境变量的值。
1. 选择 `Submit`。几秒钟后，`New Revision` 模态将关闭，新修订版本将被排队进行部署。

## 查看构建和服务器日志

每个修订版本都有构建和服务器日志。

从 `LangGraph Platform` 视图开始...

1. 从 `Revisions` 表中选择所需的修订版本。右侧会滑出一个面板，默认选择 `Build` 选项卡，显示该修订版本的构建日志。
1. 在面板中，选择 `Server` 选项卡以查看该修订版本的服务器日志。服务器日志仅在修订版本部署后可用。
1. 在 `Server` 选项卡中，根据需要调整日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为 `Last 7 days`。

## 查看部署指标

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 选择一个现有的部署进行监控。
1. 选择 `Monitoring` 选项卡以查看部署指标。请参阅[所有可用指标](../../concepts/langgraph_control_plane.md#monitoring)的列表。
1. 在 `Monitoring` 选项卡中，根据需要使用日期/时间范围选择器。默认情况下，日期/时间范围选择器设置为 `Last 15 minutes`。

## 中断修订

中断修订将停止该修订版本的部署。

!!! warning "未定义行为"
    中断的修订版本具有未定义行为。这仅在您需要部署新修订版本但已经有一个“卡住”的修订版本进行中时才有用。将来，此功能可能会被移除。

从 `LangGraph Platform` 视图开始...

1. 从 `Revisions` 表中选择所需修订版本行右侧的菜单图标（三个点）。
1. 从菜单中选择 `Interrupt`。
1. 将出现一个模态。查看确认消息。选择 `Interrupt revision`。

## 删除部署

从 <a href="https://smith.langchain.com/" target="_blank">LangSmith UI</a> 开始...

1. 在左侧导航面板中，选择 `LangGraph Platform`。`LangGraph Platform` 视图包含现有 LangGraph Platform 部署的列表。
1. 选择所需部署行右侧的菜单图标（三个点），然后选择 `Delete`。
1. 将出现一个 `Confirmation` 模态。选择 `Delete`。

## 部署设置

从 `LangGraph Platform` 视图开始...

1. 在右上角，选择齿轮图标 (`Deployment Settings`)。
1. 更新 `Git Branch` 为所需的仓库。
1. 勾选/取消勾选复选框以 `Automatically update deployment on push to branch`。
    1. 分支创建/删除和标签创建/删除事件不会触发更新。只有推送到现有分支才会触发更新。
    1. 对同一分支的连续快速推送不会触发后续更新。将来，此功能可能会被更改/改进。

## 添加或移除 GitHub 仓库

在安装并授权 LangChain 的 `hosted-langserve` GitHub 应用后，可以修改该应用的仓库访问权限以添加新仓库或移除现有仓库。如果创建了新仓库，可能需要显式添加它。

1. 从 GitHub 个人资料中，导航至 `Settings` > `Applications` > `hosted-langserve` > 点击 `Configure`。
1. 在 `Repository access` 下，选择 `All repositories` 或 `Only select repositories`。如果选择了 `Only select repositories`，则必须显式添加新仓库。
1. 点击 `Save`。
1. 创建新部署时，下拉菜单中的 GitHub 仓库列表将更新以反映仓库访问权限的更改。

## 白名单 IP 地址

2025年1月6日之后创建的 `LangGraph Platform` 部署的所有流量将通过 NAT 网关进行。
此 NAT 网关将拥有几个静态 IP 地址，具体取决于您部署所在的区域。请参阅下表获取要白名单的 IP 地址列表：

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