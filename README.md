<picture class="github-only">
  <source media="(prefers-color-scheme: light)" srcset="https://langchain-ai.github.io/langgraph/static/wordmark_dark.svg">
  <source media="(prefers-color-scheme: dark)" srcset="https://langchain-ai.github.io/langgraph/static/wordmark_light.svg">
  <img alt="LangGraph Logo" src="https://langchain-ai.github.io/langgraph/static/wordmark_dark.svg" width="80%">
</picture>

<div>
<br>
</div>

[![Version](https://img.shields.io/pypi/v/langgraph.svg)](https://pypi.org/project/langgraph/)
[![Downloads](https://static.pepy.tech/badge/langgraph/month)](https://pepy.tech/project/langgraph)
[![Open Issues](https://img.shields.io/github/issues-raw/langchain-ai/langgraph)](https://github.com/langchain-ai/langgraph/issues)
[![Docs](https://img.shields.io/badge/docs-latest-blue)](https://langchain-ai.github.io/langgraph/)

LangGraph 现已获得多家未来代理公司的信赖——包括 Klarna、Replit、Elastic 等——LangGraph 是一个低级编排框架，用于构建、管理和部署长期运行、有状态的代理。

## 入门

安装 LangGraph：

```
pip install -U langgraph
```

然后，[使用预构件组件](https://langchain-ai.github.io/langgraph/agents/agents/)创建代理：

```python
# pip install -qU "langchain[anthropic]" to call the model

from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:
    """获取给定城市的 ist ."""
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    prompt="You are a helpful assistant"
)

# 运行代理
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

更多信息，请参阅 [快速入门](https://langchain-ai.github.io/langgraph/agents/agents/)。或者，要了解如何使用可自定义的架构、长期记忆和其他复杂任务处理来构建 [代理工作流](https://langchain-ai.github.io/langgraph/concepts/low_level/)，请参阅 [LangGraph 基础教程](https://langchain-ai.github.io/langgraph/tutorials/get-started/1-build-basic-chatbot/)。

## 核心优势

LangGraph 为*任何*长期运行、有状态的工作流或代理提供底层支持基础设施。LangGraph 不会抽象化提示词或架构，并提供以下核心优势：

- [持久化执行](https://langchain-ai.github.io/langgraph/concepts/durable_execution/)：构建能够容错并可长时间运行的代理，并能自动从中断处恢复。
- [人工干预](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)：在执行的任何阶段检查和修改代理状态，从而无缝地融入人工监督。
- [全面的记忆](https://langchain-ai.github.io/langgraph/concepts/memory/)：创建真正有状态的代理，兼具用于持续推理的短期工作记忆和跨会话的长期持久化记忆。
- [通过 LangSmith 进行调试](http://www.langchain.com/langsmith)：通过可视化工具深入了解复杂的代理行为，这些工具可以追踪执行路径、捕获状态转换并提供详细的运行时指标。
- [生产级部署](https://langchain-ai.github.io/langgraph/concepts/deployment_options/)：使用专为处理有状态、长期运行的工作流的独特挑战而设计的可扩展基础设施，自信地部署复杂的代理系统。

## LangGraph 的生态系统

虽然 LangGraph 可以独立使用，但它也可以与任何 LangChain 产品无缝集成，为开发者提供一套完整的代理构建工具。为了改进你的 LLM 应用开发，请将 LangGraph 与以下产品搭配使用：

- [LangSmith](http://www.langchain.com/langsmith) — 有助于代理评估和可观察性。调试性能不佳的 LLM 应用运行、评估代理轨迹、获得生产环境的可视性，并随着时间的推移提高性能。
- [LangSmith Deployment](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) — 使用专为长期运行、有状态工作流而设计的部署平台，轻松部署和扩展代理。发现、重用、配置和跨团队共享代理——并通过 [LangGraph Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/) 进行快速可视化原型迭代。
- [LangChain](https://python.langchain.com/docs/introduction/) – 提供集成和可组合组件，以简化 LLM 应用开发。

> [!NOTE]
> 正在寻找 LangGraph 的 JavaScript 版本？请参阅 [JS 仓库](https://github.com/langchain-ai/langgraphjs) 和 [JS 文档](https://langchain-ai.github.io/langgraphjs/)。

## 附加资源

- [指南](https://langchain-ai.github.io/langgraph/guides/)：关于流式传输、添加记忆和持久化以及设计模式（例如分支、子图等）的快速、可操作的代码片段。
- [参考](https://langchain-ai.github.io/langgraph/reference/graphs/)：关于核心类、方法、如何使用图和检查点 API 以及更高级别的预构件组件的详细参考。
- [示例](https://langchain-ai.github.io/langgraph/examples/)：关于开始使用 LangGraph 的引导式示例。
- [LangChain 论坛](https://forum.langchain.com/)：与社区联系，分享你所有的技术问题、想法和反馈。
- [LangChain Academy](https://academy.langchain.com/courses/intro-to-langgraph)：在我们免费、结构化的课程中学习 LangGraph 的基础知识。
- [模板](https://langchain-ai.github.io/langgraph/concepts/template_applications/)：通用代理工作流（例如 ReAct 代理、记忆、检索等）的预构件参考应用，可以克隆和改编。
- [案例研究](https://www.langchain.com/built-with-langgraph)：听听行业领导者如何使用 LangGraph 来大规模发布 AI 应用。

## 致谢

LangGraph 的灵感来源于 [Pregel](https://research.google.com/pubs/pub37252/) 和 [Apache Beam](https://beam.apache.org/)。公共接口借鉴了 [NetworkX](https://networkx.org/documentation/latest/) 的设计。LangGraph 由 LangChain Inc.（LangChain 的创建者）构建，但可以与 LangChain 一起使用。