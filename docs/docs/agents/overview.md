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

LangGraph 提供了底层原生组件和高级预构建组件，用于构建基于 Agent 的应用程序。本节着重介绍预构建的、即用型组件，它们能帮助您快速、可靠地构建 Agent 系统——无需从头开始实现编排、记忆或人工反馈处理。

## 什么是 Agent？

一个 _Agent_ 由三个组件构成：一个 **大型语言模型 (LLM)**，它可以使用的一组 **工具**，以及一个提供指令的 **Prompt**。

LLM 在循环中运行。在每次迭代中，它会选择一个工具来调用，提供输入，接收结果（一个观察），并利用该观察来指导下一个动作。循环会一直持续，直到满足某个停止条件——通常是 Agent 收集了足够的信息来响应用户。

<figure markdown="1">
![image](./assets/agent.png){: style="max-height:400px"}
<figcaption>Agent 循环：LLM 选择要调用的工具，并使用其输出来完成用户请求。</figcaption>
</figure>

## 主要特性

LangGraph 包含了几项对于构建健壮、可用于生产的 Agent 系统至关重要的能力：

- [**集成记忆**](../how-tos/memory/add-memory.md)：原生支持 _短期_（基于会话）和 _长期_（跨会话持久化）记忆，使聊天机器人和助手能够实现有状态的行为。
- [**人工干预控制**](../concepts/human_in_the_loop.md)：执行可以 _无限期_ 暂停，以等待人工反馈——这与仅限于实时交互的、基于 WebSocket 的解决方案不同。这使得在工作流程的任何点都能实现异步批准、修正或干预。
- [**流式传输支持**](../how-tos/streaming.md)：Agent 状态、模型 Token、工具输出或组合流的实时流式传输。
- [**部署工具**](../tutorials/langgraph-platform/local-server.md)：包含无需基础设施的部署工具。[**LangGraph Platform**](https://langchain-ai.github.io/langgraph/concepts/langgraph_platform/) 支持测试、调试和部署。
  - [**Studio**](https://langchain-ai.github.io/langgraph/concepts/langgraph_studio/)：一个用于检查和调试工作流的可视化 IDE。
  - 支持多种[**部署选项**](https://langchain-ai.github.io/langgraph/concepts/deployment_options.md)以供生产使用。

## 高级构建块

LangGraph 提供了一套预构建的组件，用于实现常见的 Agent 行为和工作流。这些抽象构建在 LangGraph 框架之上，提供了更快的生产路径，同时保持了进行高级定制的灵活性。

使用 LangGraph 进行 Agent 开发，可以让您专注于应用程序的逻辑和行为，而不是构建和维护状态、内存和人工反馈的支持基础设施。

:::python

## 包生态系统

高级组件被组织到几个包中，每个包都有特定的侧重点。

| 包                                      | 描述                                                               | 安装                                    |
| --------------------------------------- | ------------------------------------------------------------------ | --------------------------------------- |
| `langgraph-prebuilt` (属于 `langgraph`) | 用于[**创建 Agent**](./agents.md)的预构建组件                       | `pip install -U langgraph langchain`    |
| `langgraph-supervisor`                  | 用于构建[**Supervisor Agent**](./multi-agent.md#supervisor)的工具 | `pip install -U langgraph-supervisor`   |
| `langgraph-swarm`                       | 用于构建[**Swarm**](./multi-agent.md#swarm)多 Agent 系统的工具     | `pip install -U langgraph-swarm`        |
| `langchain-mcp-adapters`                | 用于与[**MCP 服务器**](./mcp.md)进行工具和资源集成的接口           | `pip install -U langchain-mcp-adapters` |
| `langmem`                               | Agent 记忆管理：[**短期和长期**](../how-tos/memory/add-memory.md) | `pip install -U langmem`                |
| `agentevals`                            | 用于[**评估 Agent 性能**](./evals.md)的实用工具                     | `pip install -U agentevals`             |

## 可视化 Agent 图

使用以下工具可可视化由
@[`create_react_agent`][create_react_agent]
生成的图，并查看相应代码大纲。它允许您探索 Agent 的基础设施，具体取决于以下组件的存在：

- [`tools`](../how-tos/tool-calling.md): Agent 可以用来执行任务的工具（函数、API 或其他可调用对象）列表。
- [`pre_model_hook`](../how-tos/create-react-agent-manage-message-history.ipynb): 在调用模型之前执行的函数。可用于压缩消息或执行其他预处理任务。
- `post_model_hook`: 在调用模型之后执行的函数。可用于实现护栏、人工干预流程或其他后处理任务。
- [`response_format`](../agents/agents.md#6-configure-structured-output): 用于约束最终输出类型的结构。例如，一个 `pydantic` `BaseModel`。

<div class="agent-layout">
  <div class="agent-graph-features-container">
    <div class="agent-graph-features">
      <h3 class="agent-section-title">特性</h3>
      <label><input type="checkbox" id="tools" checked> <code>tools</code></label>
      <label><input type="checkbox" id="pre_model_hook"> <code>pre_model_hook</code></label>
      <label><input type="checkbox" id="post_model_hook"> <code>post_model_hook</code></label>
      <label><input type="checkbox" id="response_format"> <code>response_format</code></label>
    </div>
  </div>

  <div class="agent-graph-container">
    <h3 class="agent-section-title">图</h3>
    <img id="agent-graph-img" src="../assets/react_agent_graphs/0001.svg" alt="graph image" style="max-width: 100%;"/>
  </div>
</div>

以下代码片段展示了如何使用
@[`create_react_agent`][create_react_agent]
创建上述 Agent（及其底层图）：

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

  lines.push(")", "", "# Visualize the graph", "# For Jupyter or GUI environments:", "agent.get_graph().draw_mermaid_png()", "", "# To save PNG to file:", "png_data = agent.get_graph().draw_mermaid_png()", "with open(\"graph.png\", \"wb\") as f:", "    f.write(png_data)", "", "# For terminal/ASCII output:", "agent.get_graph().draw_ascii()");

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

:::

:::js

## 包生态系统

高级组件被组织到几个包中，每个包都有特定的侧重点。

| 包                       | 描述                                            | 安装                                             |
| ------------------------ | ----------------------------------------------- | ------------------------------------------------ |
| `langgraph`              | 用于[**创建 Agent**](./agents.md)的预构建组件  | `npm install @langchain/langgraph @langchain/core` |
| `langgraph-supervisor`   | 用来构建[**Supervisor Agent**](./multi-agent.md#supervisor)的工具 | `npm install @langchain/langgraph-supervisor`      |
| `langgraph-swarm`        | 用于构建[**Swarm**](./multi-agent.md#swarm)多 Agent 系统的工具 | `npm install @langchain/langgraph-swarm`           |
| `langchain-mcp-adapters` | 用于与[**MCP 服务器**](./mcp.md)集成的接口      | `npm install @langchain/mcp-adapters`              |
| `agentevals`             | 用于[**评估 Agent 性能**](./evals.md)的工具     | `npm install agentevals`                           |

## 可视化 Agent 图

使用以下工具可可视化由 @[`createReactAgent`][create_react_agent] 生成的图，并查看相应代码大纲。它允许您探索 Agent 的基础设施，具体取决于以下组件的存在：

- [`tools`](./tools.md): Agent 可以用来执行任务的工具（函数、API 或其他可调用对象）列表。
- `preModelHook`: 在调用模型之前执行的函数。可用于压缩消息或执行其他预处理任务。
- `postModelHook`: 在调用模型之后执行的函数。可用于实现护栏、人工干预流程或其他后处理任务。
- [`responseFormat`](./agents.md#6-configure-structured-output): 用于约束最终输出类型的数据结构（通过 Zod schema）。

<div class="agent-layout">
  <div class="agent-graph-features-container">
    <div class="agent-graph-features">
      <h3 class="agent-section-title">特性</h3>
      <label><input type="checkbox" id="tools" checked> <code>tools</code></label>
      <label><input type="checkbox" id="preModelHook"> <code>preModelHook</code></label>
      <label><input type="checkbox" id="postModelHook"> <code>postModelHook</code></label>
      <label><input type="checkbox" id="responseFormat"> <code>responseFormat</code></label>
    </div>
  </div>

  <div class="agent-graph-container">
    <h3 class="agent-section-title">图</h3>
    <img id="agent-graph-img" src="../assets/react_agent_graphs/0001.svg" alt="graph image" style="max-width: 100%;"/>
  </div>
</div>

以下代码片段展示了如何使用 @[`createReactAgent`][create_react_agent] 创建上述 Agent（及其底层图）：

<div class="language-typescript">
  <pre><code id="agent-code" class="language-typescript"></code></pre>
</div>

<script>
function getCheckedValue(id) {
  return document.getElementById(id).checked ? "1" : "0";
}

function getKey() {
  return [
    getCheckedValue("responseFormat"),
    getCheckedValue("postModelHook"),
    getCheckedValue("preModelHook"),
    getCheckedValue("tools")
  ].join("");
}

function dedent(strings, ...values) {
  const str = String.raw({ raw: strings }, ...values)
  const [space] = str.split("\n").filter(Boolean).at(0).match(/^(\s*)/)
  const spaceLen = space.length
  return str.split("\n").map(line => line.slice(spaceLen)).join("\n").trim()
}

Object.assign(dedent, {
  offset: (size) => (strings, ...values) => {
    return dedent(strings, ...values).split("\n").map(line => " ".repeat(size) + line).join("\n")
  }
})




function generateCodeSnippet({ tools, pre, post, response }) {
  const lines = []

  lines.push(dedent`
    import { createReactAgent } from "@langchain/langgraph/prebuilt";
    import { ChatOpenAI } from "@langchain/openai";
  `)

  if (tools) lines.push(`import { tool } from "@langchain/core/tools";`);  
  if (response || tools) lines.push(`import { z } from "zod";`);

  lines.push("", dedent`
    const agent = createReactAgent({
      llm: new ChatOpenAI({ model: "o4-mini" }),
  `)

  if (tools) {
    lines.push(dedent.offset(2)`
      tools: [
        tool(() => "Sample tool output", {
          name: "sampleTool",
          schema: z.object({}),
        }),
      ],
    `)
  }

  if (pre) {
    lines.push(dedent.offset(2)`
      preModelHook: (state) => ({ llmInputMessages: state.messages }),
    `)
  }

  if (post) {
    lines.push(dedent.offset(2)`
      postModelHook: (state) => state,
    `)
  }

  if (response) {
    lines.push(dedent.offset(2)`
      responseFormat: z.object({ result: z.string() }),
    `)
  }

  lines.push(`});`);

  return lines.join("\n");
}

function render() {
  const key = getKey();
  document.getElementById("agent-graph-img").src = `../assets/react_agent_graphs/${key}.svg`;

  const state = {
    tools: document.getElementById("tools").checked,
    pre: document.getElementById("preModelHook").checked,
    post: document.getElementById("postModelHook").checked,
    response: document.getElementById("responseFormat").checked
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

:::