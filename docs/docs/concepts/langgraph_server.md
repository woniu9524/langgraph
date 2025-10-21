---
search:
  boost: 2
---

# LangGraph Server

**LangGraph Server** 提供了一个用于创建和管理基于代理的应用程序的 API。它构建在 [assistants](assistants.md)（即为特定任务配置的代理）的概念之上，并包含内置的 [persistence](persistence.md#memory-store) 和一个**任务队列**。这个多功能的 API 支持广泛的代理应用程序用例，从后台处理到实时交互。

使用 LangGraph Server 创建和管理 [assistants](assistants.md)、[threads](./persistence.md#threads)、[runs](./assistants.md#execution)、[cron jobs](../cloud/concepts/cron_jobs.md)、[webhooks](../cloud/concepts/webhooks.md) 等。

!!! tip "API 参考"

    有关 API 端点和数据模型的详细信息，请参阅 [LangGraph Platform API 参考文档](../cloud/reference/api/api_ref.html)。

## 应用程序结构

要部署 LangGraph Server 应用程序，您需要指定要部署的图（graph），以及任何相关的配置设置，例如依赖项和环境变量。

阅读 [应用程序结构](./application_structure.md) 指南，了解如何构建 LangGraph 应用程序以进行部署。

## 部署组成部分

当您部署 LangGraph Server 时，您正在部署一个或多个 [graphs](#graphs)、一个用于 [persistence](persistence.md) 的数据库和一个任务队列。

### Graphs

当您使用 LangGraph Server 部署一个图时，您实际上是在部署一个 [Assistant](assistants.md) 的“蓝图”。

[Assistant](assistants.md) 是一个图与特定的配置设置配对。您可以为每个图创建多个 assistant，每个 assistant 具有独特的设置，以适应可以由同一图提供的不同用例。

部署后，LangGraph Server 将自动为每个图创建一个默认 assistant，使用该图的默认配置设置。

!!! note

    我们经常将一个图视为实现了 [agent](agentic_concepts.md)，但一个图不一定需要实现一个 agent。例如，一个图可以实现一个简单的
    聊天机器人，它只支持来回对话，而不能影响任何应用程序控制流。实际上，随着应用程序变得越来越复杂，一个图通常会实现一个更复杂的流程，该流程可能会使用 [多个 agents](./multi_agent.md) 协同工作。

### Persistence and task queue

LangGraph Server 利用数据库进行 [persistence](persistence.md) 和任务队列。

目前，LangGraph Server 只支持 [Postgres](https://www.postgresql.org/) 作为数据库，并支持 [Redis](https://redis.io/) 作为任务队列。

如果您使用 [LangGraph Platform](./langgraph_cloud.md) 进行部署，这些组件将由平台为您管理。如果您要在自己的基础设施上部署 LangGraph Server，则需要自行设置和管理这些组件。

请查阅 [部署选项](./deployment_options.md) 指南，了解有关这些组件如何设置和管理的更多信息。

## 了解更多

* LangGraph [应用程序结构](./application_structure.md) 指南解释了如何构建 LangGraph 应用程序以进行部署。
* [LangGraph Platform API 参考](../cloud/reference/api/api_ref.html) 提供了关于 API 端点和数据模型的详细信息。