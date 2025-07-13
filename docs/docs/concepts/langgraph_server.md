---
search:
  boost: 2
---

# LangGraph Server

**LangGraph Server** 提供了一个用于创建和管理基于代理的应用程序的 API。它构建在 [assistants](assistants.md) 的概念之上，这些 assistants 是为特定任务配置的代理，并包含内置的 [persistence](persistence.md#memory-store) 和 **task queue**。这个多功能的 API 支持广泛的代理应用程序用例，从后台处理到实时交互。

使用 LangGraph Server 来创建和管理 [assistants](assistants.md)，[threads](./persistence.md#threads)，[runs](./assistants.md#execution)，[cron jobs](../cloud/concepts/cron_jobs.md)，[webhooks](../cloud/concepts/webhooks.md) 等。

!!! tip "API 参考"
  
    有关 API 端点和数据模型的详细信息，请参阅 [LangGraph Platform API 参考文档](../cloud/reference/api/api_ref.html)。

## Server 版本

LangGraph Server 有两个版本：

- `Lite` 是 LangGraph Server 的一个有限版本，您可以在本地或自行托管（每年最多执行 100 万个 [nodes executed](../concepts/faq.md#what-does-nodes-executed-mean-for-langgraph-platform-usage)）。
- `Enterprise` 是 LangGraph Server 的完整版本。要使用 `Enterprise` 版本，您必须获取一个许可证密钥，在运行 Docker 镜像时需要指定该密钥。要获取许可证密钥，请发送电子邮件至 sales@langchain.dev。

功能差异：

|       | Lite       | Enterprise |
|-------|------------|------------|
| [Cron Jobs](../cloud/concepts/cron_jobs.md) |❌|✅|
| [Custom Authentication](../concepts/auth.md) |❌|✅|
| [Deployment options](../concepts/deployment_options.md) | 单独的容器 | 云SaaS，自行托管数据平面，自行托管控制平面，单独的容器

## 应用程序结构

要部署 LangGraph Server 应用程序，您需要指定要部署的图表以及任何相关的配置设置，例如依赖项和环境变量。

阅读 [application structure](./application_structure.md) 指南以了解如何构建 LangGraph 应用程序以进行部署。

## 部署的组成部分

当您部署 LangGraph Server 时，您将部署一个或多个 [graphs](#graphs)，一个用于 [persistence](persistence.md) 的数据库和一个任务队列。

### Graphs

当您使用 LangGraph Server 部署一个图表时，您实际上是在部署一个 [Assistant](assistants.md) 的“蓝图”。

[Assistant](assistants.md) 是一个图表与特定的配置设置配对。您可以为每个图表创建多个 assistants，每个 assistants 都具有独特的设置，以适应由同一图表提供的不同用例。

部署时，LangGraph Server 将自动为每个图表使用该图表的默认配置设置创建一个默认 assistant。

!!! note

    我们经常认为一个图表实现了一个 [agent](agentic_concepts.md)，但一个图表不一定需要实现一个 agent。例如，一个图表可以实现一个简单的
    聊天机器人，它只支持来回对话，而不能影响任何应用程序控制流。实际上，随着应用程序变得越来越复杂，一个图表通常会实现一个更复杂的流程，该流程可能会使用 [多个 agents](./multi_agent.md) 协同工作。

### Persistence and task queue

LangGraph Server 利用数据库进行 [persistence](persistence.md) 和任务队列。

目前，只有 [Postgres](https://www.postgresql.org/) 被支持作为 LangGraph Server 的数据库，[Redis](https://redis.io/) 作为任务队列。

如果您使用 [LangGraph Platform](./langgraph_cloud.md) 进行部署，则这些组件由您管理。如果您在自己的基础设施上部署 LangGraph Server，则需要自己设置和管理这些组件。

请查阅 [deployment options](./deployment_options.md) 指南以获取有关这些组件如何设置和管理的更多信息。

## 了解更多

* LangGraph [Application Structure](./application_structure.md) 指南解释了如何构建 LangGraph 应用程序以进行部署。
* [LangGraph Platform API Reference](../cloud/reference/api/api_ref.html) 提供了有关 API 端点和数据模型的详细信息。