# 调试 LangSmith 追踪

本指南将介绍如何在 LangGraph Studio 中打开 LangSmith 追踪，以进行交互式调查和调试。

## 打开已部署的追踪

1. 打开 LangSmith 追踪，并选择根运行（root run）。
2. 点击“在 Studio 中运行”（Run in Studio）。

这将打开 LangGraph Studio，并连接到相关的 LangGraph Platform 部署，同时选中该追踪的父追踪（parent thread）。

## 使用远程追踪测试本地代理

本节将介绍如何使用远程 LangSmith 追踪来测试本地代理。这使您能够将生产追踪作为本地测试的输入，从而在开发环境中调试和验证代理的修改。

### 要求

- 一个 LangSmith 追踪的线程（traced thread）
- 一个本地运行的代理。请参阅 [此处](../how-tos/studio/quick_start.md#local-development-server) 获取设置说明。

!!! info "本地代理要求"

    - langgraph>=0.3.18
    - langgraph-api>=0.0.32
    - 包含与远程追踪中相同的节点集

### 克隆追踪

1. 打开 LangSmith 追踪，并选择根运行（root run）。
2. 点击“在 Studio 中运行”（Run in Studio）旁边的下拉菜单。
3. 输入您的本地代理的 URL。
4. 选择“本地克隆追踪”（Clone thread locally）。
5. 如果存在多个图表（graphs），请选择目标图表。

将在您的本地代理中创建一个新线程，该线程的历史记录将从远程线程推断并复制，然后您将被导航到 LangGraph Studio 以查看您本地运行的应用程序。