---
title: 概述
search:
  boost: 2
tags:
  - agent
hide:
  - tags
---

# 使用预构建组件进行 Agent 开发

LangGraph 同时提供低级原语和高级预构建组件，用于构建基于 Agent 的应用程序。本节重点介绍预构建的、即用型组件，它们旨在帮助您快速可靠地构建 Agent 系统——无需从头开始实现编排、内存或人工反馈处理。

## 什么是 Agent？

*Agent* 由三个组件组成：一个 **大型语言模型 (LLM)**、一套它可以使用的 **工具** 和一个提供指令的 **提示 (prompt)**。

LLM 在循环中运行。在每次迭代中，它会选择一个工具来调用，提供输入，接收结果（一个观察），并利用该观察来指导下一个动作。循环持续进行， যতক্ষণ না একটি থামার শর্ত পূরণ হয় — সাধারণত যখন এজেন্ট পর্যাপ্ত তথ্য সংগ্রহ করে ব্যবহারকারীর কাছে প্রতিক্রিয়া জানায়।

<figure markdown="1">
![image](./assets/agent.png){: style="max-height:400px"}
<figcaption>Agent 循环：LLM 选择工具并使用它们的输出来满足用户请求。</figcaption>
</figure>

## 主要功能

LangGraph 包含构建健壮、生产就绪的 Agent 系统所需的几项功能：

- [**内存集成**](../how-tos/memory/add-memory.md)：原生支持*短期*（基于会话）和*长期*（跨会话持久化）内存，从而在聊天机器人和助手实现有状态行为。
- [**人工干预控制**](../concepts/human_in_the_loop.md)：执行可以*无限期*暂停以等待人工反馈——与仅限于实时交互的基于 WebSocket 的解决方案不同。这使得在工作流的任何点都可以进行异步批准、纠正或干预。
- [**流式支持**](../how-tos/streaming.md)：Agent 状态、模型令牌、工具输出或组合流的实时流式传输。
- [**部署工具**](../tutorials/langgraph-platform/local-server.md)：包含无基础设施的部署工具。[**LangGraph Platform**](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) 支持测试、调试和部署。
    - [**Studio**](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/)：一个用于检查和调试工作流的可视化 IDE。
    - 支持多种[**部署选项**](https://langchain-ai.github.io/langgraph/concepts/deployment_options.md)以供生产使用。

## 高级构建块

LangGraph 提供了一套预构建的组件，这些组件实现了常见的 Agent 行为和工作流。这些抽象构建在 LangGraph 框架之上，提供了更快的生产路径，同时保持了灵活度以进行高级定制。

使用 LangGraph 进行 Agent 开发，可以让您专注于应用程序的逻辑和行为，而不是构建和维护状态、内存和人工反馈的支持基础设施。

## 包生态系统

高级组件被组织到几个包中，每个包都有特定的焦点。

| 包                                     | 描述                                                              | 安装                            |
|---------------------------------------|-----------------------------------------------------------------|---------------------------------|
| `langgraph-prebuilt` (属于 `langgraph`) | 用于[**创建 Agent**](./agents.md)的预构建组件                  | `pip install -U langgraph langchain` |
| `langgraph-supervisor`                  | 用于构建[**Supervisor**](./multi-agent.md#supervisor) Agent 的工具 | `pip install -U langgraph-supervisor` |
| `langgraph-swarm`                     | 用于构建[**Swarm**](./multi-agent.md#swarm)多 Agent 系统的工具     | `pip install -U langgraph-swarm`    |
| `langchain-mcp-adapters`                | 用于工具和资源集成的[**MCP 服务器**](./mcp.md)的接口               | `pip install -U langchain-mcp-adapters` |
| `langmem`                             | Agent 内存管理：[**短期和长期**](../how-tos/memory/add-memory.md)          | `pip install -U langmem`            |
| `agentevals`                          | 用于[**评估 Agent 性能**](./evals.md)的实用工具                    | `pip install -U agentevals`         |

## 可视化 Agent Graph

使用以下工具可视化由
[`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent]
生成的图形，并查看相应代码的轮廓。
它允许您探索由以下要素定义的 Agent 基础设施：

* [`tools`](../how-tos/tool-calling.md)：Agent 可以用来执行任务的工具（函数、API 或其他可调用对象）列表。
* [`pre_model_hook`](../how-tos/create-react-agent-manage-message-history.ipynb)：在调用模型之前调用的函数。可用于压缩消息或执行其他预处理任务。
* `post_model_hook`：在调用模型之后调用的函数。可用于实现护栏、人工干预流程或其他后处理任务。
* [`response_format`](../agents/agents.md#6-configure-structured-output)：用于约束最终输出类型的数据结构，例如 `pydantic` `BaseModel`。

<div class="agent-layout">
  <div class="agent-graph-features-container">
    <div class="agent-graph-features">
      <h3 class="agent-section-title">功能</h3>
      <label><input type="checkbox" id="tools" checked> <code>tools</code></label>
      <label><input type="checkbox" id="pre_model_hook"> <code>pre_model_hook</code></label>
      <label><input type="checkbox" id="post_model_hook"> <code>post_model_hook</code></label>
      <label><input type="checkbox" id="response_format"> <code>response_format</code></label>
    </div>
  </div>

  <div class="agent-graph-container">
    <h3 class="agent-section-title">Graph</h3>
    <img id="agent-graph-img" src="../assets/react_agent_graphs/0001.svg" alt="graph image" style="max-width: 100%;"/>
  </div>
</div>


以下代码片段演示了如何使用
[`create_react_agent`][langgraph.prebuilt.chat_agent_executor.create_react_agent]
创建上述 Agent（及底层图形）：

<div class="language-python">
  <pre><code id="agent-code" class="language-python"></code></pre>
</div>


<script>
function getCheckedValue(id) {
  return document.getElementById(id).checked ? "1" : "0";
}

function getKey() {
  return [
    getCheckedValue("response_format"),
    getCheckedValue("post_model_hook"),
    getCheckedValue("pre_model_hook"),
    getCheckedValue("tools")
  ].join("");
}

function generateCodeSnippet({ tools, pre, post, response }) {
  const lines = [
    "from langgraph.prebuilt import create_react_agent",
    "from langchain_openai import ChatOpenAI"
  ];

  if (response) lines.push("from pydantic import BaseModel");

  lines.push("", 'model = ChatOpenAI("o4-mini")', "");

  if (tools) {
    lines.push(
      "def tool() -> None:",
      '    """Testing tool."""',
      "    ...",
      ""
    );
  }

  if (pre) {
    lines.push(
      "def pre_model_hook() -> None:",
      '    """Pre-model hook."""',
      "    ...",
      ""
    );
  }

  if (post) {
    lines.push(
      "def post_model_hook() -> None:",
      '    """Post-model hook."""',
      "    ...",
      ""
    );
  }

  if (response) {
    lines.push(
      "class ResponseFormat(BaseModel):",
      '    """Response format for the agent."""',
      "    result: str",
      ""
    );
  }

  lines.push("agent = create_react_agent(");
  lines.push("    model,");

  if (tools) lines.push("    tools=[tool],");
  if (pre) lines.push("    pre_model_hook=pre_model_hook,");
  if (post) lines.push("    post_model_hook=post_model_hook,");
  if (response) lines.push("    response_format=ResponseFormat,");

  lines.push(")", "", "agent.get_graph().draw_mermaid_png()");

  return lines.join("\n");
}

async function render() {
  const key = getKey();
  document.getElementById("agent-graph-img").src = `../assets/react_agent_graphs/${key}.svg`;

  const state = {
    tools: document.getElementById("tools").checked,
    pre: document.getElementById("pre_model_hook").checked,
    post: document.getElementById("post_model_hook").checked,
    response: document.getElementById("response_format").checked
  };

  document.getElementById("agent-code").textContent = generateCodeSnippet(state);
}

function initializeWidget() {
  render(); // no need for `await` here
  document.querySelectorAll(".agent-graph-features input").forEach((input) => {
    input.addEventListener("change", render);
  });
}

// Init for both full reload and SPA nav (used by MkDocs Material)
window.addEventListener("DOMContentLoaded", initializeWidget);
document$.subscribe(initializeWidget);
</script>