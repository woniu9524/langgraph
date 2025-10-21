---
search:
  boost: 2
---

# Workflows and Agents

本指南回顾了 Agentic 系统的常见模式。在描述这些系统时，区分“工作流”和“Agent”会很有用。Anthropic 的“构建有效的 Agent”博文中有对这种区别的绝佳解释：

> 工作流（Workflows）是通过预定义的代码路径来编排 LLM 和工具的系统。
> 相比之下，Agent 则是 LLM 动态地指导自身进程和工具使用，从而保持对任务完成方式的控制的系统。

这里有一种简单的方式来可视化这些差异：

![Agent Workflow](../concepts/img/agent_workflow.png)

在构建 Agent 和工作流时，LangGraph 提供了许多优势，包括持久性、流式输出以及对调试和部署的支持。

## 设置

:::python
您可以使用[任何支持结构化输出和工具调用的聊天模型](https://python.langchain.com/docs/integrations/chat/)。下面，我们将展示安装包、设置 API 密钥以及测试 Anthropic 的结构化输出/工具调用的过程。

??? "安装依赖"

    ```bash
    pip install langchain_core langchain-anthropic langgraph
    ```

初始化一个 LLM

```python
import os
import getpass

from langchain_anthropic import ChatAnthropic

def _set_env(var: str):
    if not os.environ.get(var):
        os.environ[var] = getpass.getpass(f"{var}: ")


_set_env("ANTHROPIC_API_KEY")

llm = ChatAnthropic(model="claude-3-5-sonnet-latest")
```

:::

:::js
您可以使用[任何支持结构化输出和工具调用的聊天模型](https://js.langchain.com/docs/integrations/chat/)。下面，我们将展示安装包、设置 API 密钥以及测试 Anthropic 的结构化输出/工具调用的过程。

??? "安装依赖"

    ```bash
    npm install @langchain/core @langchain/anthropic @langchain/langgraph
    ```

初始化一个 LLM

```typescript
import { ChatAnthropic } from "@langchain/anthropic";

process.env.ANTHROPIC_API_KEY = "YOUR_API_KEY";

const llm = new ChatAnthropic({ model: "claude-3-5-sonnet-latest" });
```

:::

## 构建块：增强型 LLM

LLM 增强功能支持工作流和 Agent 的构建。这些功能包括结构化输出和工具调用，如下图所示，摘自 Anthropic 关于“构建有效的 Agent”的博文：

![augmented_llm.png](./workflows/img/augmented_llm.png)

:::python

```python
# 结构化输出的模式（Schema）
from pydantic import BaseModel, Field

class SearchQuery(BaseModel):
    search_query: str = Field(None, description="用于优化网络搜索的查询。")
    justification: str = Field(
        None, description="此查询与用户请求相关的原因。"
    )


# 使用模式增强 LLM 以支持结构化输出
structured_llm = llm.with_structured_output(SearchQuery)

# 调用增强型 LLM
output = structured_llm.invoke("Calcium CT 评分与高胆固醇有什么关系？")

# 定义一个工具
def multiply(a: int, b: int) -> int:
    return a * b

# 使用工具增强 LLM
llm_with_tools = llm.bind_tools([multiply])

# 调用 LLM 并输入触发工具调用的内容
msg = llm_with_tools.invoke("2 乘以 3 是多少？")

# 获取工具调用
msg.tool_calls
```

:::

:::js

```typescript
import { z } from "zod";
import { tool } from "@langchain/core/tools";

// 结构化输出的模式（Schema）
const SearchQuery = z.object({
  search_query: z.string().describe("用于优化网络搜索的查询。"),
  justification: z
    .string()
    .describe("此查询与用户请求相关的原因。"),
});

// 使用模式增强 LLM 以支持结构化输出
const structuredLlm = llm.withStructuredOutput(SearchQuery);

// 调用增强型 LLM
const output = await structuredLlm.invoke(
  "Calcium CT 评分与高胆固醇有什么关系？"
);

// 定义一个工具
const multiply = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a * b;
  },
  {
    name: "multiply",
    description: "将两个数字相乘",
    schema: z.object({
      a: z.number(),
      b: z.number(),
    }),
  }
);

// 使用工具增强 LLM
const llmWithTools = llm.bindTools([multiply]);

// 调用 LLM 并输入触发工具调用的内容
const msg = await llmWithTools.invoke("2 乘以 3 是多少？");

// 获取工具调用
console.log(msg.tool_calls);
```

:::

## Prompt Chaining（提示链）

在提示链（Prompt Chaining）中，每一次 LLM 调用都处理前一次调用的输出。

正如 Anthropic 关于“构建有效的 Agent”的博文中所述：

> 提示链将任务分解为一系列步骤，其中每次 LLM 调用都处理前一次调用的输出。您可以对任何中间步骤添加程序化检查（参见下图中“门”（gate）），以确保流程仍在正轨上。

> 何时使用此工作流：此工作流非常适合可以轻松、干净地分解为固定子任务的情况。主要目标是通过使每次 LLM 调用都成为一个更容易的任务来权衡延迟以换取更高的准确性。

![prompt_chain.png](./workflows/img/prompt_chain.png)

=== "Graph API"

    :::python
    ```python
    from typing_extensions import TypedDict
    from langgraph.graph import StateGraph, START, END
    from IPython.display import Image, display


    # Graph 状态
    class State(TypedDict):
        topic: str
        joke: str
        improved_joke: str
        final_joke: str


    # 节点
    def generate_joke(state: State):
        """第一次 LLM 调用以生成初始笑话"""

        msg = llm.invoke(f"写一个关于 {state['topic']} 的简短笑话")
        return {"joke": msg.content}


    def check_punchline(state: State):
        """门函数，用于检查笑话是否有笑点"""

        # 简单检查——笑话是否包含“？”或“！”
        if "?" in state["joke"] or "!" in state["joke"]:
            return "Pass"
        return "Fail"


    def improve_joke(state: State):
        """第二次 LLM 调用以改进笑话"""

        msg = llm.invoke(f"通过加入双关语，让这个笑话更有趣：{state['joke']}")
        return {"improved_joke": msg.content}


    def polish_joke(state: State):
        """第三次 LLM 调用进行最终润色"""

        msg = llm.invoke(f"为这个笑话添加一个出人意料的转折：{state['improved_joke']}")
        return {"final_joke": msg.content}


    # 构建工作流
    workflow = StateGraph(State)

    # 添加节点
    workflow.add_node("generate_joke", generate_joke)
    workflow.add_node("improve_joke", improve_joke)
    workflow.add_node("polish_joke", polish_joke)

    # 添加边以连接节点
    workflow.add_edge(START, "generate_joke")
    workflow.add_conditional_edges(
        "generate_joke", check_punchline, {"Fail": "improve_joke", "Pass": END}
    )
    workflow.add_edge("improve_joke", "polish_joke")
    workflow.add_edge("polish_joke", END)

    # 编译
    chain = workflow.compile()

    # 显示工作流
    display(Image(chain.get_graph().draw_mermaid_png()))

    # 调用
    state = chain.invoke({"topic": "猫"})
    print("初始笑话:")
    print(state["joke"])
    print("\n--- --- ---\n")
    if "improved_joke" in state:
        print("改进后的笑话:")
        print(state["improved_joke"])
        print("\n--- --- ---\n")

        print("最终笑话:")
        print(state["final_joke"])
    else:
        print("笑话未能通过质量门——未检测到笑点！")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/a0281fca-3a71-46de-beee-791468607b75/r

    **资源:**

    **LangChain Academy**

    请在此处[此处](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/chain.ipynb)查看我们关于提示链的课程。
    :::

    :::js
    ```typescript
    import { StateGraph, START, END } from "@langchain/langgraph";
    import { z } from "zod";

    // Graph 状态
    const State = z.object({
      topic: z.string(),
      joke: z.string().optional(),
      improved_joke: z.string().optional(),
      final_joke: z.string().optional(),
    });

    // 节点
    const generateJoke = async (state: z.infer<typeof State>) => {
      // 第一次 LLM 调用以生成初始笑话
      const msg = await llm.invoke(`写一个关于 ${state.topic} 的简短笑话`);
      return { joke: msg.content };
    };

    const checkPunchline = (state: z.infer<typeof State>) => {
      // 门函数，用于检查笑话是否有笑点
      // 简单检查——笑话是否包含“？”或“！”
      if (state.joke && (state.joke.includes("?") || state.joke.includes("!"))) {
        return "Pass";
      }
      return "Fail";
    };

    const improveJoke = async (state: z.infer<typeof State>) => {
      // 第二次 LLM 调用以改进笑话
      const msg = await llm.invoke(`通过加入双关语，让这个笑话更有趣：${state.joke}`);
      return { improved_joke: msg.content };
    };

    const polishJoke = async (state: z.infer<typeof State>) => {
      // 第三次 LLM 调用进行最终润色
      const msg = await llm.invoke(`为这个笑话添加一个出人意料的转折：${state.improved_joke}`);
      return { final_joke: msg.content };
    };

    // 构建工作流
    const workflow = new StateGraph(State)
      .addNode("generate_joke", generateJoke)
      .addNode("improve_joke", improveJoke)
      .addNode("polish_joke", polishJoke)
      .addEdge(START, "generate_joke")
      .addConditionalEdges(
        "generate_joke",
        checkPunchline,
        { "Fail": "improve_joke", "Pass": END }
      )
      .addEdge("improve_joke", "polish_joke")
      .addEdge("polish_joke", END);

    // 编译
    const chain = workflow.compile();

    // 显示工作流
    import * as fs from "node:fs/promises";
    const drawableGraph = await chain.getGraphAsync();
    const image = await drawableGraph.drawMermaidPng();
    const imageBuffer = new Uint8Array(await image.arrayBuffer());
    await fs.writeFile("workflow.png", imageBuffer);

    // 调用
    const state = await chain.invoke({ topic: "猫" });
    console.log("初始笑话:");
    console.log(state.joke);
    console.log("\n--- --- ---\n");
    if (state.improved_joke) {
      console.log("改进后的笑话:");
      console.log(state.improved_joke);
      console.log("\n--- --- ---\n");

      console.log("最终笑话:");
      console.log(state.final_joke);
    } else {
      console.log("笑话未能通过质量门——未检测到笑点！");
    }
    ```
    :::

=== "Functional API"

    :::python
    ```python
    from langgraph.func import entrypoint, task


    # 任务
    @task
    def generate_joke(topic: str):
        """第一次 LLM 调用以生成初始笑话"""
        msg = llm.invoke(f"写一个关于 {topic} 的简短笑话")
        return msg.content


    def check_punchline(joke: str):
        """门函数，用于检查笑话是否有笑点"""
        # 简单检查——笑话是否包含“？”或“！”
        if "?" in joke or "!" in joke:
            return "Fail"

        return "Pass"


    @task
    def improve_joke(joke: str):
        """第二次 LLM 调用以改进笑话"""
        msg = llm.invoke(f"通过加入双关语，让这个笑话更有趣：{joke}")
        return msg.content


    @task
    def polish_joke(joke: str):
        """第三次 LLM 调用进行最终润色"""
        msg = llm.invoke(f"为这个笑话添加一个出人意料的转折：{joke}")
        return msg.content


    @entrypoint()
    def prompt_chaining_workflow(topic: str):
        original_joke = generate_joke(topic).result()
        if check_punchline(original_joke) == "Pass":
            return original_joke

        improved_joke = improve_joke(original_joke).result()
        return polish_joke(improved_joke).result()

    # 调用
    for step in prompt_chaining_workflow.stream("猫", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/332fa4fc-b6ca-416e-baa3-161625e69163/r
    :::

    :::js
    ```typescript
    import { entrypoint, task } from "@langchain/langgraph";

    // 任务
    const generateJoke = task("generate_joke", async (topic: string) => {
      // 第一次 LLM 调用以生成初始笑话
      const msg = await llm.invoke(`写一个关于 ${topic} 的简短笑话`);
      return msg.content;
    });

    const checkPunchline = (joke: string) => {
      // 门函数，用于检查笑话是否有笑点
      // 简单检查——笑话是否包含“？”或“！”
      if (joke.includes("?") || joke.includes("!")) {
        return "Pass";
      }
      return "Fail";
    };

    const improveJoke = task("improve_joke", async (joke: string) => {
      // 第二次 LLM 调用以改进笑话
      const msg = await llm.invoke(`通过加入双关语，让这个笑话更有趣：${joke}`);
      return msg.content;
    });

    const polishJoke = task("polish_joke", async (joke: string) => {
      // 第三次 LLM 调用进行最终润色
      const msg = await llm.invoke(`为这个笑话添加一个出人意料的转折：${joke}`);
      return msg.content;
    });

    const promptChainingWorkflow = entrypoint("promptChainingWorkflow", async (topic: string) => {
      const originalJoke = await generateJoke(topic);
      if (checkPunchline(originalJoke) === "Pass") {
        return originalJoke;
      }

      const improvedJoke = await improveJoke(originalJoke);
      return await polishJoke(improvedJoke);
    });

    // 调用
    const stream = await promptChainingWorkflow.stream("猫", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## Parallelization（并行化）

通过并行化，LLM 可以同时处理一个任务：

> LLM 有时可以同时处理一个任务，并通过程序化方式聚合它们的输出。此工作流（并行化）有两种主要变体：分段（Sectioning）：将任务分解为并行运行的独立子任务。投票（Voting）：多次运行同一个任务以获得不同的输出。

> 何时使用此工作流：当可分解的子任务可以并行化以提高速度，或者当需要多个视角或尝试来获得更高置信度的结果时，并行化非常有效。对于涉及多种考虑因素的复杂任务，LLM 通常在每个考虑因素都由一个单独的 LLM 调用处理时表现更好，从而可以专注于每个特定方面。

![parallelization.png](./workflows/img/parallelization.png)

=== "Graph API"

    :::python
    ```python
    # Graph 状态
    class State(TypedDict):
        topic: str
        joke: str
        story: str
        poem: str
        combined_output: str


    # 节点
    def call_llm_1(state: State):
        """第一次 LLM 调用以生成初始笑话"""

        msg = llm.invoke(f"写一个关于 {state['topic']} 的笑话")
        return {"joke": msg.content}


    def call_llm_2(state: State):
        """第二次 LLM 调用以生成故事”"""

        msg = llm.invoke(f"写一个关于 {state['topic']} 的故事")
        return {"story": msg.content}


    def call_llm_3(state: State):
        """第三次 LLM 调用以生成诗歌”"""

        msg = llm.invoke(f"写一首关于 {state['topic']} 的诗")
        return {"poem": msg.content}


    def aggregator(state: State):
        """将笑话和故事合并成一个输出”"""

        combined = f"这是关于 {state['topic']} 的一个故事、笑话和诗！\n\n"
        combined += f"故事:\n{state['story']}\n\n"
        combined += f"笑话:\n{state['joke']}\n\n"
        combined += f"诗歌:\n{state['poem']}"
        return {"combined_output": combined}


    # 构建工作流
    parallel_builder = StateGraph(State)

    # 添加节点
    parallel_builder.add_node("call_llm_1", call_llm_1)
    parallel_builder.add_node("call_llm_2", call_llm_2)
    parallel_builder.add_node("call_llm_3", call_llm_3)
    parallel_builder.add_node("aggregator", aggregator)

    # 添加边以连接节点
    parallel_builder.add_edge(START, "call_llm_1")
    parallel_builder.add_edge(START, "call_llm_2")
    parallel_builder.add_edge(START, "call_llm_3")
    parallel_builder.add_edge("call_llm_1", "aggregator")
    parallel_builder.add_edge("call_llm_2", "aggregator")
    parallel_builder.add_edge("call_llm_3", "aggregator")
    parallel_builder.add_edge("aggregator", END)
    parallel_workflow = parallel_builder.compile()

    # 显示工作流
    display(Image(parallel_workflow.get_graph().draw_mermaid_png()))

    # 调用
    state = parallel_workflow.invoke({"topic": "猫"})
    print(state["combined_output"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/3be2e53c-ca94-40dd-934f-82ff87fac277/r

    **资源:**

    **文档**

    请在此处[此处](https://langchain-ai.github.io/langgraph/how-tos/branching/)查看我们关于并行化的文档。

    **LangChain Academy**

    请在此处[此处](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/simple-graph.ipynb)查看我们关于并行化的课程。
    :::

    :::js
    ```typescript
    // Graph 状态
    const State = z.object({
      topic: z.string(),
      joke: z.string().optional(),
      story: z.string().optional(),
      poem: z.string().optional(),
      combined_output: z.string().optional(),
    });

    // 节点
    const callLlm1 = async (state: z.infer<typeof State>) => {
      // 第一次 LLM 调用以生成初始笑话
      const msg = await llm.invoke(`写一个关于 ${state.topic} 的笑话`);
      return { joke: msg.content };
    };

    const callLlm2 = async (state: z.infer<typeof State>) => {
      // 第二次 LLM 调用以生成故事”
      const msg = await llm.invoke(`写一个关于 ${state.topic} 的故事`);
      return { story: msg.content };
    };

    const callLlm3 = async (state: z.infer<typeof State>) => {
      // 第三次 LLM 调用以生成诗歌”
      const msg = await llm.invoke(`写一首关于 ${state.topic} 的诗`);
      return { poem: msg.content };
    };

    const aggregator = (state: z.infer<typeof State>) => {
      // 将笑话和故事合并成一个输出”
      let combined = `这是关于 ${state.topic} 的一个故事、笑话和诗！\n\n`;
      combined += `故事:\n${state.story}\n\n`;
      combined += `笑话:\n${state.joke}\n\n`;
      combined += `诗歌:\n${state.poem}`;
      return { combined_output: combined };
    };

    // 构建工作流
    const parallelBuilder = new StateGraph(State)
      .addNode("call_llm_1", callLlm1)
      .addNode("call_llm_2", callLlm2)
      .addNode("call_llm_3", callLlm3)
      .addNode("aggregator", aggregator)
      .addEdge(START, "call_llm_1")
      .addEdge(START, "call_llm_2")
      .addEdge(START, "call_llm_3")
      .addEdge("call_llm_1", "aggregator")
      .addEdge("call_llm_2", "aggregator")
      .addEdge("call_llm_3", "aggregator")
      .addEdge("aggregator", END);

    const parallelWorkflow = parallelBuilder.compile();

    // 调用
    const state = await parallelWorkflow.invoke({ topic: "猫" });
    console.log(state.combined_output);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    @task
    def call_llm_1(topic: str):
        """第一次 LLM 调用以生成初始笑话"""
        msg = llm.invoke(f"写一个关于 {topic} 的笑话")
        return msg.content


    @task
    def call_llm_2(topic: str):
        """第二次 LLM 调用以生成故事”"""
        msg = llm.invoke(f"写一个关于 {topic} 的故事")
        return msg.content


    @task
    def call_llm_3(topic):
        """第三次 LLM 调用以生成诗歌”"""
        msg = llm.invoke(f"写一首关于 {topic} 的诗")
        return msg.content


    @task
    def aggregator(topic, joke, story, poem):
        """将笑话和故事合并成一个输出”"""

        combined = f"这是关于 {topic} 的一个故事、笑话和诗！\n\n"
        combined += f"故事:\n{story}\n\n"
        combined += f"笑话:\n{joke}\n\n"
        combined += f"诗歌:\n{poem}"
        return combined


    # 构建工作流
    @entrypoint()
    def parallel_workflow(topic: str):
        joke_fut = call_llm_1(topic)
        story_fut = call_llm_2(topic)
        poem_fut = call_llm_3(topic)
        return aggregator(
            topic, joke_fut.result(), story_fut.result(), poem_fut.result()
        ).result()

    # 调用
    for step in parallel_workflow.stream("猫", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/623d033f-e814-41e9-80b1-75e6abb67801/r
    :::

    :::js
    ```typescript
    const callLlm1 = task("call_llm_1", async (topic: string) => {
      // 第一次 LLM 调用以生成初始笑话
      const msg = await llm.invoke(`写一个关于 ${topic} 的笑话`);
      return msg.content;
    });

    const callLlm2 = task("call_llm_2", async (topic: string) => {
      // 第二次 LLM 调用以生成故事”
      const msg = await llm.invoke(`写一个关于 ${topic} 的故事`);
      return msg.content;
    });

    const callLlm3 = task("call_llm_3", async (topic: string) => {
      // 第三次 LLM 调用以生成诗歌”
      const msg = await llm.invoke(`写一首关于 ${topic} 的诗`);
      return msg.content;
    });

    const aggregator = task("aggregator", (topic: string, joke: string, story: string, poem: string) => {
      // 将笑话和故事合并成一个输出”
      let combined = `这是关于 ${topic} 的一个故事、笑话和诗！\n\n`;
      combined += `故事:\n${story}\n\n`;
      combined += `笑话:\n${joke}\n\n`;
      combined += `诗歌:\n${poem}`;
      return combined;
    });

    // 构建工作流
    const parallelWorkflow = entrypoint("parallelWorkflow", async (topic: string) => {
      const jokeFut = callLlm1(topic);
      const storyFut = callLlm2(topic);
      const poemFut = callLlm3(topic);

      return await aggregator(
        topic,
        await jokeFut,
        await storyFut,
        await poemFut
      );
    });

    // 调用
    const stream = await parallelWorkflow.stream("猫", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## Routing（路由）

路由根据输入对内容进行分类，并将其定向到后续任务。正如 Anthropic 关于“构建有效的 Agent”的博文中所述：

> 路由（Routing）对输入进行分类，并将其定向到专门的后续任务。此工作流允许关注点分离，并构建更专业的提示。没有此工作流，针对一种输入的优化会损害对其他输入的性能。

> 何时使用此工作流：路由适用于可以准确分类（由 LLM 或更传统的分类模型/算法）并且处理起来有明显区别的复杂任务。

![routing.png](./workflows/img/routing.png)

=== "Graph API"

    :::python
    ```python
    from typing_extensions import Literal
    from langchain_core.messages import HumanMessage, SystemMessage


    # 用于路由逻辑的结构化输出模式
    class Route(BaseModel):
        step: Literal["poem", "story", "joke"] = Field(
            None, description="路由过程的下一步"
        )


    # 使用模式增强 LLM 以支持结构化输出
    router = llm.with_structured_output(Route)


    # 状态
    class State(TypedDict):
        input: str
        decision: str
        output: str


    # 节点
    def llm_call_1(state: State):
        """写一个故事”"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_2(state: State):
        """写一个笑话”"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_3(state: State):
        """写一首诗”"""

        result = llm.invoke(state["input"])
        return {"output": result.content}


    def llm_call_router(state: State):
        """将输入路由到相应的节点"""

        # 运行增强型 LLM 并输出结构化结果，用作路由逻辑
        decision = router.invoke(
            [
                SystemMessage(
                    content="根据用户请求将输入路由到故事、笑话或诗歌。"
                ),
                HumanMessage(content=state["input"]),
            ]
        )

        return {"decision": decision.step}


    # 用于将输入路由到相应节点的条件边函数
    def route_decision(state: State):
        # 返回要访问的下一个节点的名称
        if state["decision"] == "story":
            return "llm_call_1"
        elif state["decision"] == "joke":
            return "llm_call_2"
        elif state["decision"] == "poem":
            return "llm_call_3"


    # 构建工作流
    router_builder = StateGraph(State)

    # 添加节点
    router_builder.add_node("llm_call_1", llm_call_1)
    router_builder.add_node("llm_call_2", llm_call_2)
    router_builder.add_node("llm_call_3", llm_call_3)
    router_builder.add_node("llm_call_router", llm_call_router)

    # 添加边以连接节点
    router_builder.add_edge(START, "llm_call_router")
    router_builder.add_conditional_edges(
        "llm_call_router",
        route_decision,
        {  # route_decision 返回的名称 : 要访问的下一个节点名称
            "llm_call_1": "llm_call_1",
            "llm_call_2": "llm_call_2",
            "llm_call_3": "llm_call_3",
        },
    )
    router_builder.add_edge("llm_call_1", END)
    router_builder.add_edge("llm_call_2", END)
    router_builder.add_edge("llm_call_3", END)

    # 编译工作流
    router_workflow = router_builder.compile()

    # 显示工作流
    display(Image(router_workflow.get_graph().draw_mermaid_png()))

    # 调用
    state = router_workflow.invoke({"input": "写一个关于猫的笑话"})
    print(state["output"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/c4580b74-fe91-47e4-96fe-7fac598d509c/r

    **资源:**

    **LangChain Academy**

    请在此处[此处](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/router.ipynb)查看我们关于路由的课程。

    **示例**

    [此处](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag_local/)是一个路由问题的 RAG 工作流。请观看我们的视频[此处](https://www.youtube.com/watch?v=bq1Plo2RhYI)。
    :::

    :::js
    ```typescript
    import { SystemMessage, HumanMessage } from "@langchain/core/messages";

    // 用于路由逻辑的结构化输出模式
    const Route = z.object({
      step: z.enum(["poem", "story", "joke"]).describe("路由过程的下一步"),
    });

    // 使用模式增强 LLM 以支持结构化输出
    const router = llm.withStructuredOutput(Route);

    // 状态
    const State = z.object({
      input: z.string(),
      decision: z.string().optional(),
      output: z.string().optional(),
    });

    // 节点
    const llmCall1 = async (state: z.infer<typeof State>) => {
      // 写一个故事”
      const result = await llm.invoke(state.input);
      return { output: result.content };
    };

    const llmCall2 = async (state: z.infer<typeof State>) => {
      // 写一个笑话”
      const result = await llm.invoke(state.input);
      return { output: result.content };
    };

    const llmCall3 = async (state: z.infer<typeof State>) => {
      // 写一首诗”
      const result = await llm.invoke(state.input);
      return { output: result.content };
    };

    const llmCallRouter = async (state: z.infer<typeof State>) => {
      // 将输入路由到相应节点
      const decision = await router.invoke([
        new SystemMessage("根据用户请求将输入路由到故事、笑话或诗歌。"),
        new HumanMessage(state.input),
      ]);

      return { decision: decision.step };
    };

    // 用于将输入路由到相应节点的条件边函数
    const routeDecision = (state: z.infer<typeof State>) => {
      // 返回要访问的下一个节点的名称
      if (state.decision === "story") {
        return "llm_call_1";
      } else if (state.decision === "joke") {
        return "llm_call_2";
      } else if (state.decision === "poem") {
        return "llm_call_3";
      }
    };

    // 构建工作流
    const routerBuilder = new StateGraph(State)
      .addNode("llm_call_1", llmCall1)
      .addNode("llm_call_2", llmCall2)
      .addNode("llm_call_3", llmCall3)
      .addNode("llm_call_router", llmCallRouter)
      .addEdge(START, "llm_call_router")
      .addConditionalEdges(
        "llm_call_router",
        routeDecision,
        {
          "llm_call_1": "llm_call_1",
          "llm_call_2": "llm_call_2",
          "llm_call_3": "llm_call_3",
        }
      )
      .addEdge("llm_call_1", END)
      .addEdge("llm_call_2", END)
      .addEdge("llm_call_3", END);

    const routerWorkflow = routerBuilder.compile();

    // 调用
    const state = await routerWorkflow.invoke({ input: "写一个关于猫的笑话" });
    console.log(state.output);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    from typing_extensions import Literal
    from pydantic import BaseModel
    from langchain_core.messages import HumanMessage, SystemMessage


    # 用于路由逻辑的结构化输出模式
    class Route(BaseModel):
        step: Literal["poem", "story", "joke"] = Field(
            None, description="路由过程的下一步"
        )


    # 使用模式增强 LLM 以支持结构化输出
    router = llm.with_structured_output(Route)


    @task
    def llm_call_1(input_: str):
        """写一个故事”"""
        result = llm.invoke(input_)
        return result.content


    @task
    def llm_call_2(input_: str):
        """写一个笑话”"""
        result = llm.invoke(input_)
        return result.content


    @task
    def llm_call_3(input_: str):
        """写一首诗”"""
        result = llm.invoke(input_)
        return result.content


    def llm_call_router(input_: str):
        """将输入路由到相应节点"""
        # 运行增强型 LLM 并输出结构化结果，用作路由逻辑
        decision = router.invoke(
            [
                SystemMessage(
                    content="根据用户请求将输入路由到故事、笑话或诗歌。"
                ),
                HumanMessage(content=input_),
            ]
        )
        return decision.step


    # 创建工作流
    @entrypoint()
    def router_workflow(input_: str):
        next_step = llm_call_router(input_)
        if next_step == "story":
            llm_call = llm_call_1
        elif next_step == "joke":
            llm_call = llm_call_2
        elif next_step == "poem":
            llm_call = llm_call_3

        return llm_call(input_).result()

    # 调用
    for step in router_workflow.stream("写一个关于猫的笑话", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/5e2eb979-82dd-402c-b1a0-a8cceaf2a28a/r
    :::

    :::js
    ```typescript
    import { SystemMessage, HumanMessage } from "@langchain/core/messages";

    // 用于路由逻辑的结构化输出模式
    const Route = z.object({
      step: z.enum(["poem", "story", "joke"]).describe(
        "路由过程的下一步"
      ),
    });

    // 使用模式增强 LLM 以支持结构化输出
    const router = llm.withStructuredOutput(Route);

    const llmCall1 = task("llm_call_1", async (input: string) => {
      // 写一个故事”
      const result = await llm.invoke(input);
      return result.content;
    });

    const llmCall2 = task("llm_call_2", async (input: string) => {
      // 写一个笑话”
      const result = await llm.invoke(input);
      return result.content;
    });

    const llmCall3 = task("llm_call_3", async (input: string) => {
      // 写一首诗”
      const result = await llm.invoke(input);
      return result.content;
    });

    const llmCallRouter = async (input: string) => {
      // 将输入路由到相应节点
      const decision = await router.invoke([
        new SystemMessage("根据用户请求将输入路由到故事、笑话或诗歌。"),
        new HumanMessage(input),
      ]);
      return decision.step;
    };

    // 创建工作流
    const routerWorkflow = entrypoint("routerWorkflow", async (input: string) => {
      const nextStep = await llmCallRouter(input);

      let llmCall: typeof llmCall1;
      if (nextStep === "story") {
        llmCall = llmCall1;
      } else if (nextStep === "joke") {
        llmCall = llmCall2;
      } else if (nextStep === "poem") {
        llmCall = llmCall3;
      }

      return await llmCall(input);
    });

    // 调用
    const stream = await routerWorkflow.stream("写一个关于猫的笑话", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## Orchestrator-Worker（协调器-工作器）

在协调器-工作器（Orchestrator-Worker）模式中，一个协调器 LLM 将任务分解，并将每个子任务委托给工作器 LLM。正如 Anthropic 关于“构建有效的 Agent”的博文中所述：

> 在协调器-工作器（Orchestrator-Workers）工作流中，一个中央 LLM 动态地分解任务，将其委托给工作器 LLM，并综合它们的输出。

> 何时使用此工作流：此工作流非常适合您无法预测所需子任务数量的复杂任务（例如，在代码中，需要更改的文件数量以及每个文件中的更改性质可能取决于任务）。虽然在拓扑上相似，但与并行化（parallelization）的关键区别在于其灵活性——子任务不是预定义的，而是由协调器根据特定输入确定的。

![worker.png](./workflows/img/worker.png)

=== "Graph API"

    :::python
    ```python
    from typing import Annotated, List
    import operator


    # 用于规划的结构化输出模式
    class Section(BaseModel):
        name: str = Field(
            description="报告这一部分的名称。",
        )
        description: str = Field(
            description="将要涵盖的主要主题和概念的简要概述。",
        )


    class Sections(BaseModel):
        sections: List[Section] = Field(
            description="报告的各个部分。",
        )


    # 使用模式增强 LLM 以支持规划
    planner = llm.with_structured_output(Sections)
    ```

    **在 LangGraph 中创建工作器**

    由于协调器-工作器工作流很常见，LangGraph **提供了 `Send` API 来支持此功能**。它允许您动态创建工作器节点，并将每个节点输入特定的输入。每个工作器都有自己的状态，所有工作器的输出都写入一个*共享状态键*，该键可供协调器图访问。这使得协调器可以访问所有工作器的输出，并允许它们将这些输出综合成最终输出。正如您下面所见，我们迭代各部分列表并将它们“发送”给工作器节点。请在此处[此处](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 和[此处](https://langchain-ai.github.io/langgraph/concepts/low_level/#send) 参阅更多文档。

    ```python
    from langgraph.types import Send


    # Graph 状态
    class State(TypedDict):
        topic: str  # 报告主题
        sections: list[Section]  # 报告的部分列表
        completed_sections: Annotated[
            list, operator.add
        ]  # 所有工作器并行写入此键
        final_report: str  # 最终报告


    # 工作器状态
    class WorkerState(TypedDict):
        section: Section
        completed_sections: Annotated[list, operator.add]


    # 节点
    def orchestrator(state: State):
        """生成报告计划的协调器”"""

        # 生成查询
        report_sections = planner.invoke(
            [
                SystemMessage(content="生成报告的计划。"),
                HumanMessage(content=f"这是报告主题：{state['topic']}"),
            ]
        )

        return {"sections": report_sections.sections}


    def llm_call(state: WorkerState):
        """工作器编写报告的一部分"""

        # 生成部分
        section = llm.invoke(
            [
                SystemMessage(
                    content="根据提供的名称和描述编写报告的一部分。每部分都不要加前导文字。使用 Markdown 格式。"
                ),
                HumanMessage(
                    content=f"这是部分名称：{state['section'].name} 和描述：{state['section'].description}"
                ),
            ]
        )

        # 将更新后的部分写入已完成的部分
        return {"completed_sections": [section.content]}


    def synthesizer(state: State):
        """从各部分合成完整报告”"""

        # 完成的部分列表
        completed_sections = state["completed_sections"]

        # 将完成的部分格式化为字符串，作为最终部分的上下文使用
        completed_report_sections = "\n\n---\n\n".join(completed_sections)

        return {"final_report": completed_report_sections}


    # 用于创建将为报告的每个部分编写内容的 llm_call 工作器的条件边函数
    def assign_workers(state: State):
        """为计划中的每个部分分派一个工作器”"""

        # 通过 Send() API 并行启动节的编写
        return [Send("llm_call", {"section": s}) for s in state["sections"]]


    # 构建工作流
    orchestrator_worker_builder = StateGraph(State)

    # 添加节点
    orchestrator_worker_builder.add_node("orchestrator", orchestrator)
    orchestrator_worker_builder.add_node("llm_call", llm_call)
    orchestrator_worker_builder.add_node("synthesizer", synthesizer)

    # 添加边以连接节点
    orchestrator_worker_builder.add_edge(START, "orchestrator")
    orchestrator_worker_builder.add_conditional_edges(
        "orchestrator", assign_workers, ["llm_call"]
    )
    orchestrator_worker_builder.add_edge("llm_call", "synthesizer")
    orchestrator_worker_builder.add_edge("synthesizer", END)

    # 编译工作流
    orchestrator_worker = orchestrator_worker_builder.compile()

    # 显示工作流
    display(Image(orchestrator_worker.get_graph().draw_mermaid_png()))

    # 调用
    state = orchestrator_worker.invoke({"topic": "创建关于 LLM 缩放定律的报告"})

    from IPython.display import Markdown
    Markdown(state["final_report"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/78cbcfc3-38bf-471d-b62a-b299b144237d/r

    **资源:**

    **LangChain Academy**

    请在此处[此处](https://github.com/langchain-ai/langchain-academy/blob/main/module-4/map-reduce.ipynb)查看我们关于协调器-工作器的课程。

    **示例**

    [此处](https://github.com/langchain-ai/report-mAIstro)是一个使用协调器-工作器进行报告规划和编写的项目。请观看我们的视频[此处](https://www.youtube.com/watch?v=wSxZ7yFbbas)。
    :::

    :::js
    ```typescript
    import "@langchain/langgraph/zod";

    // 用于规划的结构化输出模式
    const Section = z.object({
      name: z.string().describe("报告这一部分的名称。"),
      description: z.string().describe("将要涵盖的主要主题和概念的简要概述。"),
    });

    const Sections = z.object({
      sections: z.array(Section).describe("报告的各个部分。"),
    });

    // 使用模式增强 LLM 以支持规划
    const planner = llm.withStructuredOutput(Sections);
    ```

    **在 LangGraph 中创建工作器**

    由于协调器-工作器工作流很常见，LangGraph **提供了 `Send` API 来支持此功能**。它允许您动态创建工作器节点，并将每个节点输入特定的输入。每个工作器都有自己的状态，所有工作器的输出都写入一个*共享状态键*，该键可供协调器图访问。这使得协调器可以访问所有工作器的输出，并允许它们将这些输出综合成最终输出。正如您下面所见，我们迭代各部分列表并将它们“发送”给工作器节点。请在此处[此处](https://langchain-ai.github.io/langgraph/how-tos/map-reduce/) 和[此处](https://langchain-ai.github.io/langgraph/concepts/low_level/#send) 参阅更多文档。

    ```typescript
    import { withLangGraph } from "@langchain/langgraph/zod";
    import { Send } from "@langchain/langgraph";

    // Graph 状态
    const State = z.object({
      topic: z.string(), // 报告主题
      sections: z.array(Section).optional(), // 报告的部分列表
      // 所有工作器都写入此键
      completed_sections: withLangGraph(z.array(z.string()), {
        reducer: {
          fn: (x, y) => x.concat(y),
        },
        default: () => [],
      }),
      final_report: z.string().optional(), // 最终报告
    });

    // 工作器状态
    const WorkerState = z.object({
      section: Section,
      completed_sections: withLangGraph(z.array(z.string()), {
        reducer: {
          fn: (x, y) => x.concat(y),
        },
        default: () => [],
      }),
    });

    // 节点
    const orchestrator = async (state: z.infer<typeof State>) => {
      // 生成报告计划的协调器”
      const reportSections = await planner.invoke([
        new SystemMessage("生成报告的计划。"),
        new HumanMessage(`这是报告主题：${state.topic}`),
      ]);

      return { sections: reportSections.sections };
    };

    const llmCall = async (state: z.infer<typeof WorkerState>) => {
      // 工作器编写报告的一部分
      const section = await llm.invoke([
        new SystemMessage(
          "根据提供的名称和描述编写报告的一部分。每部分都不要加前导文字。使用 Markdown 格式。"
        ),
        new HumanMessage(
          `这是部分名称：${state.section.name} 和描述：${state.section.description}`
        ),
      ]);

      // 将更新后的部分写入已完成的部分
      return { completed_sections: [section.content] };
    };

    const synthesizer = (state: z.infer<typeof State>) => {
      // 从各部分合成完整报告”
      const completedSections = state.completed_sections;
      const completedReportSections = completedSections.join("\n\n---\n\n");
      return { final_report: completedReportSections };
    };

    // 用于创建 llm_call 工作器的条件边函数
    const assignWorkers = (state: z.infer<typeof State>) => {
      // 为计划中的每个部分分派一个工作器”
      return state.sections!.map((s) => new Send("llm_call", { section: s }));
    };

    // 构建工作流
    const orchestratorWorkerBuilder = new StateGraph(State)
      .addNode("orchestrator", orchestrator)
      .addNode("llm_call", llmCall)
      .addNode("synthesizer", synthesizer)
      .addEdge(START, "orchestrator")
      .addConditionalEdges("orchestrator", assignWorkers, ["llm_call"])
      .addEdge("llm_call", "synthesizer")
      .addEdge("synthesizer", END);

    // 编译工作流
    const orchestratorWorker = orchestratorWorkerBuilder.compile();

    // 调用
    const state = await orchestratorWorker.invoke({ topic: "创建关于 LLM 缩放定律的报告" });
    console.log(state.final_report);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    from typing import List


    # 用于规划的结构化输出模式
    class Section(BaseModel):
        name: str = Field(
            description="报告这一部分的名称。",
        )
        description: str = Field(
            description="将要涵盖的主要主题和概念的简要概述。",
        )


    class Sections(BaseModel):
        sections: List[Section] = Field(
            description="报告的各个部分。",
        )


    # 使用模式增强 LLM 以支持规划
    planner = llm.with_structured_output(Sections)


    @task
    def orchestrator(topic: str):
        """生成报告计划的协调器”"""
        # 生成查询
        report_sections = planner.invoke(
            [
                SystemMessage(content="生成报告的计划。"),
                HumanMessage(content=f"这是报告主题：{topic}"),
            ]
        )

        return report_sections.sections


    @task
    def llm_call(section: Section):
        """工作器编写报告的一部分”"""

        # 生成部分
        result = llm.invoke(
            [
                SystemMessage(content="编写报告的一部分。"),
                HumanMessage(
                    content=f"这是部分名称：{section.name} 和描述：{section.description}"
                ),
            ]
        )

        # 将更新后的部分写入已完成的部分
        return result.content


    @task
    def synthesizer(completed_sections: list[str]):
        """从各部分合成完整报告”"""
        final_report = "\n\n---\n\n".join(completed_sections)
        return final_report


    @entrypoint()
    def orchestrator_worker(topic: str):
        sections = orchestrator(topic).result()
        section_futures = [llm_call(section) for section in sections]
        final_report = synthesizer(
            [section_fut.result() for section_fut in section_futures]
        ).result()
        return final_report

    # 调用
    report = orchestrator_worker.invoke("创建关于 LLM 缩放定律的报告")
    from IPython.display import Markdown
    Markdown(report)
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/75a636d0-6179-4a12-9836-e0aa571e87c5/r
    :::

    :::js
    ```typescript
    // 用于规划的结构化输出模式
    const Section = z.object({
      name: z.string().describe("报告这一部分的名称。"),
      description: z.string().describe("将要涵盖的主要主题和概念的简要概述。"),
    });

    const Sections = z.object({
      sections: z.array(Section).describe("报告的各个部分。"),
    });

    // 使用模式增强 LLM 以支持规划
    const planner = llm.withStructuredOutput(Sections);

    const orchestrator = task("orchestrator", async (topic: string) => {
      // 生成报告计划的协调器”
      const reportSections = await planner.invoke([
        new SystemMessage("生成报告的计划。"),
        new HumanMessage(`这是报告主题：${topic}`),
      ]);
      return reportSections.sections;
    });

    const llmCall = task("llm_call", async (section: z.infer<typeof Section>) => {
      // 工作器编写报告的一部分
      const result = await llm.invoke([
        new SystemMessage("编写报告的一部分。"),
        new HumanMessage(
          `这是部分名称：${section.name} 和描述：${section.description}`
        ),
      ]);
      return result.content;
    });

    const synthesizer = task("synthesizer", (completedSections: string[]) => {
      // 从各部分合成完整报告”
      const finalReport = completedSections.join("\n\n---\n\n");
      return finalReport;
    });

    const orchestratorWorker = entrypoint("orchestratorWorker", async (topic: string) => {
      const sections = await orchestrator(topic);
      const sectionFutures = sections.map((section) => llmCall(section));
      const finalReport = await synthesizer(
        await Promise.all(sectionFutures)
      );
      return finalReport;
    });

    // 调用
    const report = await orchestratorWorker.invoke("创建关于 LLM 缩放定律的报告");
    console.log(report);
    ```
    :::

## Evaluator-Optimizer（评估器-优化器）

在评估器-优化器（Evaluator-Optimizer）工作流中，一个 LLM 调用生成响应，而另一个 LLM 调用在循环中提供评估和反馈：

> 何时使用此工作流：此工作流在我们拥有清晰的评估标准，并且迭代改进能带来可衡量价值时特别有效。良好契合的两个迹象是：首先，当人类表达反馈时，LLM 的响应可以得到可衡量的改进；其次，LLM 可以提供这种反馈。这类似于人类作家在撰写一份精炼文档时可能会经历的迭代写作过程。

![evaluator_optimizer.png](./workflows/img/evaluator_optimizer.png)

=== "Graph API"

    :::python
    ```python
    # Graph 状态
    class State(TypedDict):
        joke: str
        topic: str
        feedback: str
        funny_or_not: str


    # 用于评估的结构化输出模式
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="决定笑话是否有趣。",
        )
        feedback: str = Field(
            description="如果笑话不有趣，请提供改进建议。",
        )


    # 使用模式增强 LLM 以支持结构化输出
    evaluator = llm.with_structured_output(Feedback)


    # 节点
    def llm_call_generator(state: State):
        """LLM 生成一个笑话”"""

        if state.get("feedback"):
            msg = llm.invoke(
                f"写一个关于 {state['topic']} 的笑话，并考虑以下反馈：{state['feedback']}"
            )
        else:
            msg = llm.invoke(f"写一个关于 {state['topic']} 的笑话")
        return {"joke": msg.content}


    def llm_call_evaluator(state: State):
        """LLM 评估笑话”"""

        grade = evaluator.invoke(f"给笑话 {state['joke']} 打分")
        return {"funny_or_not": grade.grade, "feedback": grade.feedback}


    # 用于根据评估器的反馈将流程路由回笑话生成器或结束的条件边函数
    def route_joke(state: State):
        """根据评估器的反馈将流程路由回笑话生成器或结束”"""

        if state["funny_or_not"] == "funny":
            return "Accepted"
        elif state["funny_or_not"] == "not funny":
            return "Rejected + Feedback"


    # 构建工作流
    optimizer_builder = StateGraph(State)

    # 添加节点
    optimizer_builder.add_node("llm_call_generator", llm_call_generator)
    optimizer_builder.add_node("llm_call_evaluator", llm_call_evaluator)

    # 添加边以连接节点
    optimizer_builder.add_edge(START, "llm_call_generator")
    optimizer_builder.add_edge("llm_call_generator", "llm_call_evaluator")
    optimizer_builder.add_conditional_edges(
        "llm_call_evaluator",
        route_joke,
        {  # route_joke 返回的名称 : 要访问的下一个节点名称
            "Accepted": END,
            "Rejected + Feedback": "llm_call_generator",
        },
    )

    # 编译工作流
    optimizer_workflow = optimizer_builder.compile()

    # 显示工作流
    display(Image(optimizer_workflow.get_graph().draw_mermaid_png()))

    # 调用
    state = optimizer_workflow.invoke({"topic": "猫"})
    print(state["joke"])
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/86ab3e60-2000-4bff-b988-9b89a3269789/r

    **资源:**

    **示例**

    [此处](https://github.com/langchain-ai/local-deep-researcher)是一个使用评估器-优化器来改进报告的助手。请观看我们的视频[此处](https://www.youtube.com/watch?v=XGuTzHoqlj8)。

    [此处](https://langchain-ai.github.io/langgraph/tutorials/rag/langgraph_adaptive_rag_local/)是一个对答案进行评分以检测幻觉或错误的 RAG 工作流。请观看我们的视频[此处](https://www.youtube.com/watch?v=bq1Plo2RhYI)。
    :::

    :::js
    ```typescript
    // Graph 状态
    const State = z.object({
      joke: z.string().optional(),
      topic: z.string(),
      feedback: z.string().optional(),
      funny_or_not: z.string().optional(),
    });

    // 用于评估的结构化输出模式
    const Feedback = z.object({
      grade: z.enum(["funny", "not funny"]).describe("决定笑话是否有趣。"),
      feedback: z.string().describe("如果笑话不有趣，请提供改进建议。"),
    });

    // 使用模式增强 LLM 以支持结构化输出
    const evaluator = llm.withStructuredOutput(Feedback);

    // 节点
    const llmCallGenerator = async (state: z.infer<typeof State>) => {
      // LLM 生成一个笑话”
      let msg;
      if (state.feedback) {
        msg = await llm.invoke(
          `写一个关于 ${state.topic} 的笑话，并考虑以下反馈：${state.feedback}`
        );
      } else {
        msg = await llm.invoke(`写一个关于 ${state.topic} 的笑话`);
      }
      return { joke: msg.content };
    };

    const llmCallEvaluator = async (state: z.infer<typeof State>) => {
      // LLM 评估笑话”
      const grade = await evaluator.invoke(`给笑话 ${state.joke} 打分`);
      return { funny_or_not: grade.grade, feedback: grade.feedback };
    };

    // 用于将流程路由回笑话生成器或结束的条件边函数
    const routeJoke = (state: z.infer<typeof State>) => {
      // 根据评估器的反馈将流程路由回笑话生成器或结束”
      if (state.funny_or_not === "funny") {
        return "Accepted";
      } else if (state.funny_or_not === "not funny") {
        return "Rejected + Feedback";
      }
    };

    // 构建工作流
    const optimizerBuilder = new StateGraph(State)
      .addNode("llm_call_generator", llmCallGenerator)
      .addNode("llm_call_evaluator", llmCallEvaluator)
      .addEdge(START, "llm_call_generator")
      .addEdge("llm_call_generator", "llm_call_evaluator")
      .addConditionalEdges(
        "llm_call_evaluator",
        routeJoke,
        {
          "Accepted": END,
          "Rejected + Feedback": "llm_call_generator",
        }
      );

    // 编译工作流
    const optimizerWorkflow = optimizerBuilder.compile();

    // 调用
    const state = await optimizerWorkflow.invoke({ topic: "猫" });
    console.log(state.joke);
    ```
    :::

=== "Functional API"

    :::python
    ```python
    # 用于评估的结构化输出模式
    class Feedback(BaseModel):
        grade: Literal["funny", "not funny"] = Field(
            description="决定笑话是否有趣。",
        )
        feedback: str = Field(
            description="如果笑话不有趣，请提供改进建议。",
        )


    # 使用模式增强 LLM 以支持结构化输出
    evaluator = llm.with_structured_output(Feedback)


    # 节点
    @task
    def llm_call_generator(topic: str, feedback: Feedback):
        """LLM 生成一个笑话”"""
        if feedback:
            msg = llm.invoke(
                f"写一个关于 {topic} 的笑话，并考虑以下反馈：{feedback}"
            )
        else:
            msg = llm.invoke(f"写一个关于 {topic} 的笑话")
        return msg.content


    @task
    def llm_call_evaluator(joke: str):
        """LLM 评估笑话”"""
        feedback = evaluator.invoke(f"给笑话 {joke} 打分")
        return feedback


    @entrypoint()
    def optimizer_workflow(topic: str):
        feedback = None
        while True:
            joke = llm_call_generator(topic, feedback).result()
            feedback = llm_call_evaluator(joke).result()
            if feedback.grade == "funny":
                break

        return joke

    # 调用
    for step in optimizer_workflow.stream("猫", stream_mode="updates"):
        print(step)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/f66830be-4339-4a6b-8a93-389ce5ae27b4/r
    :::

    :::js
    ```typescript
    // 用于评估的结构化输出模式
    const Feedback = z.object({
      grade: z.enum(["funny", "not funny"]).describe("决定笑话是否有趣。"),
      feedback: z.string().describe("如果笑话不有趣，请提供改进建议。"),
    });

    // 使用模式增强 LLM 以支持结构化输出
    const evaluator = llm.withStructuredOutput(Feedback);

    // 节点
    const llmCallGenerator = task("llm_call_generator", async (topic: string, feedback?: string) => {
      // LLM 生成一个笑话”
      if (feedback) {
        const msg = await llm.invoke(
          `写一个关于 ${topic} 的笑话，并考虑以下反馈：${feedback}`
        );
        return msg.content;
      } else {
        const msg = await llm.invoke(`写一个关于 ${topic} 的笑话`);
        return msg.content;
      }
    });

    const llmCallEvaluator = task("llm_call_evaluator", async (joke: string) => {
      // LLM 评估笑话”
      const feedback = await evaluator.invoke(`给笑话 ${joke} 打分`);
      return feedback;
    });

    const optimizerWorkflow = entrypoint("optimizerWorkflow", async (topic: string) => {
      let feedback;
      while (true) {
        const joke = await llmCallGenerator(topic, feedback?.feedback);
        feedback = await llmCallEvaluator(joke);
        if (feedback.grade === "funny") {
          return joke;
        }
      }
    });

    // 调用
    const stream = await optimizerWorkflow.stream("猫", { streamMode: "updates" });
    for await (const step of stream) {
      console.log(step);
      console.log("\n");
    }
    ```
    :::

## Agent

Agent 通常实现为 LLM 在循环中基于环境反馈执行操作（通过工具调用）。正如 Anthropic 关于“构建有效的 Agent”的博文中所述：

> Agent 可以处理复杂的任务，但其实际实现通常很简单。它们通常只是 LLM 在循环中根据环境反馈使用工具。因此，清晰周到地设计工具集及其文档至关重要。

> 何时使用 Agent：Agent 可用于开放式问题，这些问题难以或不可能预测所需的步骤数，并且您无法硬编码固定路径。LLM 可能会运行很多轮，您必须对其决策能力有一定的信任。Agent 的自主性使其成为在受信任环境中扩展任务的理想选择。

![agent.png](./workflows/img/agent.png)

:::python

```python
from langchain_core.tools import tool


# 定义工具
@tool
def multiply(a: int, b: int) -> int:
    """将 a 和 b 相乘。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a * b


@tool
def add(a: int, b: int) -> int:
    """将 a 和 b 相加。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a + b


@tool
def divide(a: int, b: int) -> float:
    """将 a 除以 b。

    Args:
        a: 第一个整数
        b: 第二个整数
    """
    return a / b


# 使用工具增强 LLM
tools = [add, multiply, divide]
tools_by_name = {tool.name: tool for tool in tools}
llm_with_tools = llm.bind_tools(tools)
```

:::

:::js

```typescript
import { tool } from "@langchain/core/tools";

// 定义工具
const multiply = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a * b;
  },
  {
    name: "multiply",
    description: "将 a 和 b 相乘。",
    schema: z.object({
      a: z.number().describe("第一个整数"),
      b: z.number().describe("第二个整数"),
    }),
  }
);

const add = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a + b;
  },
  {
    name: "add",
    description: "将 a 和 b 相加。",
    schema: z.object({
      a: z.number().describe("第一个整数"),
      b: z.number().describe("第二个整数"),
    }),
  }
);

const divide = tool(
  async ({ a, b }: { a: number; b: number }) => {
    return a / b;
  },
  {
    name: "divide",
    description: "将 a 除以 b。",
    schema: z.object({
      a: z.number().describe("第一个整数"),
      b: z.number().describe("第二个整数"),
    }),
  }
);

// 使用工具增强 LLM
const tools = [add, multiply, divide];
const toolsByName = Object.fromEntries(tools.map((tool) => [tool.name, tool]));
const llmWithTools = llm.bindTools(tools);
```

:::

=== "Graph API"

    :::python
    ```python
    from langgraph.graph import MessagesState
    from langchain_core.messages import SystemMessage, HumanMessage, ToolMessage


    # 节点
    def llm_call(state: MessagesState):
        """LLM 决定是否调用工具”"""

        return {
            "messages": [
                llm_with_tools.invoke(
                    [
                        SystemMessage(
                            content="你是一个乐于助人的助手，负责对一组输入执行算术运算。"
                        )
                    ]
                    + state["messages"]
                )
            ]
        }


    def tool_node(state: dict):
        """执行工具调用”"""

        result = []
        for tool_call in state["messages"][-1].tool_calls:
            tool = tools_by_name[tool_call["name"]]
            observation = tool.invoke(tool_call["args"])
            result.append(ToolMessage(content=observation, tool_call_id=tool_call["id"]))
        return {"messages": result}


    # 用于根据 LLM 是否进行了工具调用来路由到工具节点或结束的条件边函数
    def should_continue(state: MessagesState) -> Literal["Action", END]:
        """根据 LLM 是否进行了工具调用来决定是继续循环还是停止”"""

        messages = state["messages"]
        last_message = messages[-1]
        # 如果 LLM 进行了工具调用，则执行操作
        if last_message.tool_calls:
            return "Action"
        # 否则，停止（回复用户）
        return END


    # 构建工作流
    agent_builder = StateGraph(MessagesState)

    # 添加节点
    agent_builder.add_node("llm_call", llm_call)
    agent_builder.add_node("environment", tool_node)

    # 添加边以连接节点
    agent_builder.add_edge(START, "llm_call")
    agent_builder.add_conditional_edges(
        "llm_call",
        should_continue,
        {
            # should_continue 返回的名称 : 要访问的下一个节点名称
            "Action": "environment",
            END: END,
        },
    )
    agent_builder.add_edge("environment", "llm_call")

    # 编译 Agent
    agent = agent_builder.compile()

    # 显示 Agent
    display(Image(agent.get_graph(xray=True).draw_mermaid_png()))

    # 调用
    messages = [HumanMessage(content="将 3 和 4 相加。")]
    messages = agent.invoke({"messages": messages})
    for m in messages["messages"]:
        m.pretty_print()
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/051f0391-6761-4f8c-a53b-22231b016690/r

    **资源:**

    **LangChain Academy**

    请在此处[此处](https://github.com/langchain-ai/langchain-academy/blob/main/module-1/agent.ipynb)查看我们关于 Agent 的课程。

    **示例**

    [此处](https://github.com/langchain-ai/memory-agent)是一个使用工具调用 Agent 来创建/存储长期记忆的项目。
    :::

    :::js
    ```typescript
    import { MessagesZodState, ToolNode } from "@langchain/langgraph/prebuilt";
    import { SystemMessage, HumanMessage, ToolMessage, isAIMessage } from "@langchain/core/messages";

    // 节点
    const llmCall = async (state: z.infer<typeof MessagesZodState>) => {
      // LLM 决定是否调用工具”
      const response = await llmWithTools.invoke([
        new SystemMessage(
          "你是一个乐于助人的助手，负责对一组输入执行算术运算。"
        ),
        ...state.messages,
      ]);
      return { messages: [response] };
    };

    const toolNode = new ToolNode(tools);

    // 用于将流量路由到工具节点或结束的条件边函数
    const shouldContinue = (state: z.infer<typeof MessagesZodState>) => {
      // 决定是继续循环还是停止”
      const messages = state.messages;
      const lastMessage = messages[messages.length - 1];
      // 如果 LLM 进行了工具调用，则执行操作
      if (isAIMessage(lastMessage) && lastMessage.tool_calls?.length) {
        return "Action";
      }
      // 否则，停止（回复用户）
      return END;
    };

    // 构建工作流
    const agentBuilder = new StateGraph(MessagesZodState)
      .addNode("llm_call", llmCall)
      .addNode("environment", toolNode)
      .addEdge(START, "llm_call")
      .addConditionalEdges(
        "llm_call",
        shouldContinue,
        {
          "Action": "environment",
          [END]: END,
        }
      )
      .addEdge("environment", "llm_call");

    // 编译 Agent
    const agent = agentBuilder.compile();

    // 调用
    const messages = [new HumanMessage("将 3 和 4 相加。")];
    const result = await agent.invoke({ messages });
    for (const m of result.messages) {
      console.log(`${m.getType()}: ${m.content}`);
    }
    ```
    :::

=== "Functional API"

    :::python
    ```python
    from langgraph.graph import add_messages
    from langchain_core.messages import (
        SystemMessage,
        HumanMessage,
        BaseMessage,
        ToolCall,
    )


    @task
    def call_llm(messages: list[BaseMessage]):
        """LLM 决定是否调用工具”"""
        return llm_with_tools.invoke(
            [
                SystemMessage(
                    content="你是一个乐于助人的助手，负责对一组输入执行算术运算。"
                )
            ]
            + messages
        )


    @task
    def call_tool(tool_call: ToolCall):
        """执行工具调用”"""
        tool = tools_by_name[tool_call["name"]]
        return tool.invoke(tool_call)


    @entrypoint()
    def agent(messages: list[BaseMessage]):
        llm_response = call_llm(messages).result()

        while True:
            if not llm_response.tool_calls:
                break

            # 执行工具
            tool_result_futures = [
                call_tool(tool_call) for tool_call in llm_response.tool_calls
            ]
            tool_results = [fut.result() for fut in tool_result_futures]
            messages = add_messages(messages, [llm_response, *tool_results])
            llm_response = call_llm(messages).result()

        messages = add_messages(messages, llm_response)
        return messages

    # 调用
    messages = [HumanMessage(content="将 3 和 4 相加。")]
    for chunk in agent.stream(messages, stream_mode="updates"):
        print(chunk)
        print("\n")
    ```

    **LangSmith Trace**

    https://smith.langchain.com/public/42ae8bf9-3935-4504-a081-8ddbcbfc8b2e/r
    :::

    :::js
    ```typescript
    import { addMessages } from "@langchain/langgraph";
    import {
      SystemMessage,
      HumanMessage,
      BaseMessage,
      ToolCall,
    } from "@langchain/core/messages";

    const callLlm = task("call_llm", async (messages: BaseMessage[]) => {
      // LLM 决定是否调用工具”
      return await llmWithTools.invoke([
        new SystemMessage(
          "你是一个乐于助人的助手，负责对一组输入执行算术运算。"
        ),
        ...messages,
      ]);
    });

    const callTool = task("call_tool", async (toolCall: ToolCall) => {
      // 执行工具调用”
      const tool = toolsByName[toolCall.name];
      return await tool.invoke(toolCall);
    });

    const agent = entrypoint("agent", async (messages: BaseMessage[]) => {
      let currentMessages = messages;
      let llmResponse = await callLlm(currentMessages);

      while (true) {
        if (!llmResponse.tool_calls?.length) {
          break;
        }

        // 执行工具
        const toolResults = await Promise.all(
          llmResponse.tool_calls.map((toolCall) => callTool(toolCall))
        );

        // 添加到消息列表
        currentMessages = addMessages(currentMessages, [
          llmResponse,
          ...toolResults,
        ]);

        // 再次调用模型
        llmResponse = await callLlm(currentMessages);
      }

      return llmResponse;
    });

    // 调用
    const messages = [new HumanMessage("将 3 和 4 相加。")];
    const stream = await agent.stream(messages, { streamMode: "updates" });
    for await (const chunk of stream) {
      console.log(chunk);
      console.log("\n");
    }
    ```
    :::

#### Pre-built（预构建）

:::python
LangGraph 还提供了一个**预构建方法**来创建如上定义的 Agent（使用 @[`create_react_agent`][create_react_agent] 函数）：

https://langchain-ai.github.io/langgraph/how-tos/create-react-agent/

```python
from langgraph.prebuilt import create_react_agent

# 传入：
# (1) 带有工具的增强型 LLM
# (2) 工具列表（用于创建工具节点）
pre_built_agent = create_react_agent(llm, tools=tools)

# 显示 Agent
display(Image(pre_built_agent.get_graph().draw_mermaid_png()))

# 调用
messages = [HumanMessage(content="将 3 和 4 相加。")]
messages = pre_built_agent.invoke({"messages": messages})
for m in messages["messages"]:
    m.pretty_print()
```

**LangSmith Trace**

https://smith.langchain.com/public/abab6a44-29f6-4b97-8164-af77413e494d/r
:::

:::js
LangGraph 还提供了一个**预构建方法**来创建如上定义的 Agent（使用 @[`createReactAgent`][create_react_agent] 函数）：

```typescript
import { createReactAgent } from "@langchain/langgraph/prebuilt";

// 传入：
// (1) 带有工具的增强型 LLM
// (2) 工具列表（用于创建工具节点）
const preBuiltAgent = createReactAgent({ llm, tools });

// 调用
const messages = [new HumanMessage("将 3 和 4 相加。")];
const result = await preBuiltAgent.invoke({ messages });
for (const m of result.messages) {
  console.log(`${m.getType()}: ${m.content}`);
}
```

:::

## LangGraph 提供的功能

通过在 LangGraph 中构建上述所有内容，我们可以获得一些优势：

### Persistence（持久性）：人工干预（Human-in-the-Loop）

LangGraph 的持久性层支持操作的中断和审批（例如，人工干预）。请参阅 LangChain Academy 的[第三模块](https://github.com/langchain-ai/langchain-academy/tree/main/module-3)。

### Persistence（持久性）：Memory（记忆）

LangGraph 的持久性层支持对话（短期）记忆和长期记忆。请参阅 LangChain Academy 的[第 2 模块](https://github.com/langchain-ai/langchain-academy/tree/main/module-2) [和第 5 模块](https://github.com/langchain-ai/langchain-academy/tree/main/module-5)：

### Streaming（流式输出）

LangGraph 提供了多种流式传输工作流/Agent 输出或中间状态的方法。请参阅 LangChain Academy 的[第 3 模块](https://github.com/langchain-ai/langchain-academy/blob/main/module-3/streaming-interruption.ipynb)。

### Deployment（部署）

LangGraph 为部署、可观测性和评估提供了一个简单的入口。请参阅 LangChain Academy 的[第 6 模块](https://github.com/langchain-ai/langchain-academy/tree/main/module-6)。