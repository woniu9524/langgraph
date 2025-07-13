# 设置 LangGraph.js 应用程序

必须使用 [LangGraph 配置文件](../reference/cli.md#configuration-file) 来配置 [LangGraph.js](https://langchain-ai.github.io/langgraphjs/) 应用程序，以便将其部署到 LangGraph 平台（或进行自托管）。本指南将讨论使用 `package.json` 指定项目依赖项来设置 LangGraph.js 应用程序进行部署的基本步骤。

本教程基于 [此存储库](https://github.com/langchain-ai/langgraphjs-studio-starter)，您可以尝试使用它来详细了解如何设置 LangGraph 应用程序以进行部署。

最终的存储库结构将如下所示：

```bash
my-app/
├── src # 所有项目代码都在这里
│   ├── utils # 图的可选工具
│   │   ├── tools.ts # 图的工具
│   │   ├── nodes.ts # 图的节点函数
│   │   └── state.ts # 图的状态定义
│   └── agent.ts # 构建图的代码
├── package.json # 包依赖项
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

在每一步之后，都会提供一个示例文件目录，以演示代码的组织方式。

## 指定依赖项

可以在 `package.json` 中指定依赖项。如果不存在这些文件中的任何一个，则可以在后面的 [LangGraph 配置文件](#create-langgraph-api-config) 中指定依赖项。

示例 `package.json` 文件：

```json
{
  "name": "langgraphjs-studio-starter",
  "packageManager": "yarn@1.22.22",
  "dependencies": {
    "@langchain/community": "^0.2.31",
    "@langchain/core": "^0.2.31",
    "@langchain/langgraph": "^0.2.0",
    "@langchain/openai": "^0.2.8"
  }
}
```

部署应用程序时，将使用您选择的包管理器安装依赖项，前提是它们符合下面列出的兼容版本范围：

```
"@langchain/core": "^0.3.42",
"@langchain/langgraph": "^0.2.57",
"@langchain/langgraph-checkpoint": "~0.0.16",
```

示例文件目录：

```bash
my-app/
└── package.json # 包依赖项
```

## 指定环境变量

可以在一个文件中（例如 `.env`）指定环境变量。请参阅 [环境变量参考](../reference/env_var.md) 以为部署配置其他变量。

示例 `.env` 文件：

```
MY_ENV_VAR_1=foo
MY_ENV_VAR_2=bar
OPENAI_API_KEY=key
TAVILY_API_KEY=key_2
```

示例文件目录：

```bash
my-app/
├── package.json
└── .env # 环境变量
```

## 定义图

实现您的图！图可以定义在单个文件中，也可以定义在多个文件中。请记下每个已编译图的变量名，以便在创建 [LangGraph 配置文件](../reference/cli.md#configuration-file) 时包含它们。

下面是一个 `agent.ts` 的示例：

```ts
import type { AIMessage } from "@langchain/core/messages";
import { TavilySearchResults } from "@langchain/community/tools/tavily_search";
import { ChatOpenAI } from "@langchain/openai";

import { MessagesAnnotation, StateGraph } from "@langchain/langgraph";
import { ToolNode } from "@langchain/langgraph/prebuilt";

const tools = [new TavilySearchResults({ maxResults: 3 })];

// 定义调用模型的函数
async function callModel(state: typeof MessagesAnnotation.State) {
  /**
   * 调用支持我们代理的 LLM。
   * 请随时自定义提示、模型和其他逻辑！
   */
  const model = new ChatOpenAI({
    model: "gpt-4o",
  }).bindTools(tools);

  const response = await model.invoke([
    {
      role: "system",
      content: `您是一个有用的助手。当前日期是 ${new Date().getTime()}。`,
    },
    ...state.messages,
  ]);

  // MessagesAnnotation 支持返回单个消息或消息数组
  return { messages: response };
}

// 定义确定是否继续的函数
function routeModelOutput(state: typeof MessagesAnnotation.State) {
  const messages = state.messages;
  const lastMessage: AIMessage = messages[messages.length - 1];
  // 如果 LLM 正在调用工具，则路由到那里。
  if ((lastMessage?.tool_calls?.length ?? 0) > 0) {
    return "tools";
  }
  // 否则结束图。
  return "__end__";
}

// 定义一个新图。
// 有关定义自定义图状态的更多信息，请参阅 https://langchain-ai.github.io/langgraphjs/how-tos/define-state/#getting-started
const workflow = new StateGraph(MessagesAnnotation)
  // 定义我们将循环的两个节点
  .addNode("callModel", callModel)
  .addNode("tools", new ToolNode(tools))
  // 将入口点设置为 `callModel`
  // 这意味着该节点是第一个被调用的节点
  .addEdge("__start__", "callModel")
  .addConditionalEdges(
    // 首先，我们定义边的源节点。我们使用 `callModel`。
    // 这意味着这些是调用 `callModel` 节点后采用的边。
    "callModel",
    // 接下来，我们传入将确定目标节点（将在调用源节点后调用）的函数。
    routeModelOutput,
    // 条件边可以路由到的可能目标列表。
    // 对于条件边正确呈现图到 Studio 是必需的
    ["tools", "__end__"]
  )
  // 这意味着在调用 `tools` 后，下一个调用的是 `callModel` 节点。
  .addEdge("tools", "callModel");

// 最后，我们将其编译！
// 这将其编译成一个可以调用和部署的图。
export const graph = workflow.compile();
```

示例文件目录：

```bash
my-app/
├── src # 所有项目代码都在这里
│   ├── utils # 图的可选工具
│   │   ├── tools.ts # 图的工具
│   │   ├── nodes.ts # 图的节点函数
│   │   └── state.ts # 图的状态定义
│   └── agent.ts # 构建图的代码
├── package.json # 包依赖项
├── .env # 环境变量
└── langgraph.json # LangGraph 的配置文件
```

## 创建 LangGraph API 配置

创建一个名为 `langgraph.json` 的 [LangGraph 配置文件](../reference/cli.md#configuration-file)。有关配置文件中每个键的详细说明，请参阅 [LangGraph 配置文件参考](../reference/cli.md#configuration-file)。

示例 `langgraph.json` 文件：

```json
{
  "node_version": "20",
  "dockerfile_lines": [],
  "dependencies": ["."],
  "graphs": {
    "agent": "./src/agent.ts:graph"
  },
  "env": ".env"
}
```

请注意，`CompiledGraph` 的变量名出现在顶级 `graphs` 键的每个子键的值的末尾（即 `:<variable_name>`）。

!!! info "配置位置"

    LangGraph 配置文件必须放置在与包含已编译图以及相关依赖项的 TypeScript 文件相同级别或更高目录中。

## 下一步

设置好项目并将其放入 GitHub 存储库后，就可以[部署您的应用程序](./cloud.md)了。