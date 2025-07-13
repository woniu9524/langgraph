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

LangGraph 受到许多塑造代理未来公司的信赖——包括 Klarna、Replit、Elastic 等——它是一个低级别协调框架，用于构建、管理和部署长期运行、有状态的代理。

## 开始使用

安装 LangGraph：

```
pip install -U langgraph
```

然后，[使用预构建组件](https://langchain-ai.github.io/langgraph/agents/agents/) 创建一个代理：

```python
# pip install -qU "langchain[anthropic]" to call the model

from langgraph.prebuilt import create_react_agent

def get_weather(city: str) -> str:
    """Get weather for a given city."""
    # **此处是注释，应翻译**
    return f"It's always sunny in {city}!"

agent = create_react_agent(
    model="anthropic:claude-3-7-sonnet-latest",
    tools=[get_weather],
    prompt="You are a helpful assistant"
)

# Run the agent
agent.invoke(
    {"messages": [{"role": "user", "content": "what is the weather in sf"}]}
)
```

有关更多信息，请参阅[快速入门](https://langchain-ai.github.io/langgraph/agents/agents/)。或者，要了解如何使用可自定义的架构、长期记忆和其他复杂任务处理来构建[代理工作流](https://langchain-ai.github.io/langgraph/concepts/low_level/)，请参阅[LangGraph 基础教程](https://langchain-ai.github.io/langgraph/tutorials/get-started/1-build-basic-chatbot/)。

## 核心优势

LangGraph 为*任何*长期运行、有状态的工作流或代理提供低级别支持基础设施。LangGraph 不会抽象化提示或架构，并提供以下中心优势：

- [持久化执行](https://langchain-ai.github.io/langgraph/concepts/durable_execution/)：构建能够从故障中恢复并在长时间内运行的代理，自动从中断处精确恢复。
- [人工干预](https://langchain-ai.github.io/langgraph/concepts/human_in_the_loop/)：通过在执行的任何阶段检查和修改代理状态，无缝地融入人工监督。
- [全面的记忆](https://langchain-ai.github.io/langgraph/concepts/memory/)：创建真正的有状态代理，兼具正在进行的推理的短期工作记忆和跨会话的长期持久记忆。
- [通过 LangSmith 进行调试](http://www.langchain.com/langsmith)：借助可视化工具深入了解复杂的代理行为，这些工具可以追踪执行路径、捕获状态转换并提供详细的运行时指标。
- [生产就绪的部署](https://langchain-ai.github.io/langgraph/concepts/deployment_options/)：使用专为有状态、长期运行的工作流的独特挑战而设计的可伸缩基础设施，自信地部署复杂的代理系统。

## LangGraph 的生态系统

虽然 LangGraph 可以独立使用，但它也可以与任何 LangChain 产品无缝集成，为开发人员提供一套完整的代理构建工具。为了改进您的 LLM 应用程序开发，请将 LangGraph 与以下产品配合使用：

- [LangSmith](http://www.langchain.com/langsmith) — 有助于代理评估和可观察性。调试性能不佳的 LLM 应用运行、评估代理轨迹、获得生产环境的可视性，并随着时间的推移提高性能。
- [LangGraph Platform](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) — 使用专用的部署平台轻松部署和扩展代理，该平台适用于长期运行的有状态工作流。跨团队发现、重用、配置和共享代理——并通过[LangGraph Studio](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/) 中的可视化原型快速迭代。
- [LangChain](https://python.langchain.com/docs/introduction/) – 提供集成和可组合组件，以简化 LLM 应用程序开发。

> [!NOTE]
> 正在寻找 LangGraph 的 JS 版本？请参阅[JS 仓库](https://github.com/langchain-ai/langgraphjs)和[JS 文档](https://langchain-ai.github.io/langgraphjs/)。

## 附加资源

- [指南](https://langchain-ai.github.io/langgraph/how-tos/)：针对流式传输、添加记忆和持久化以及设计模式（例如分支、子图等）等主题的快速、可操作的代码片段。
- [参考](https://langchain-ai.github.io/langgraph/reference/graphs/)：有关核心类、方法、如何使用图和检查点 API 以及更高级的预构建组件的详细参考。
- [示例](https://langchain-ai.github.io/langgraph/tutorials/overview/)：关于开始使用 LangGraph 的指导性示例。
- [LangChain Academy](https://academy.langchain.com/courses/intro-to-langgraph)：在我们免费、结构化的课程中学习 LangGraph 的基础知识。
- [模板](https://langchain-ai.github.io/langgraph/concepts/template_applications/)：常见代理工作流（例如 ReAct 代理、记忆、检索等）的预构建参考应用程序，可以克隆和改编。
- [案例研究](https://www.langchain.com/built-with-langgraph)：了解行业领导者如何使用 LangGraph 大规模发布 AI 应用。

## 致谢

LangGraph 的灵感来源于 [Pregel](https://research.google.com/pubs/pub37252/) 和 [Apache Beam](https://beam.apache.org/)。公开接口借鉴了 [NetworkX](https://networkx.org/documentation/latest/) 的灵感。LangGraph 由 LangChain 的创建者 LangChain Inc. 构建，但也可在不使用 LangChain 的情况下使用。