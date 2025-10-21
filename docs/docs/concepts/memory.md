---
search:
  boost: 2
---

# Memory

[Memory](../how-tos/memory/add-memory.md) 是一个用于记忆先前交互信息的系统。对于 AI Agent 来说，Memory（记忆）至关重要，因为它能让 Agent 记住之前的交互，从反馈中学习，并适应用户的偏好。随着 Agent 承担更复杂的任务和更多用户交互，此功能对于效率和用户满意度都至关重要。

本概念指南将根据其回忆范围，介绍两种类型的 Memory：

- [Short-term memory](#short-term-memory)（短期记忆），也称为 [thread](persistence.md#threads)（会话）范围的 Memory，通过维护会话中的消息历史来跟踪当前的对话。LangGraph 将 short-term memory 作为 agent [state](low_level.md#state)（状态）的一部分进行管理。State 会使用 [checkpointer](persistence.md#checkpoints) 持久化到数据库，以便随时恢复 thread。当 graph 被调用或一个 step 完成时，short-term memory 会更新，并且 State 会在每个 step 开始时读取。

- [Long-term memory](#long-term-memory)（长期记忆）跨会话存储用户特定或应用级别的数据，并在不同的 conversational threads（对话线索）之间共享。它可以_随时_在_任何 thread_中回忆。Memory 的作用域可以限定在任何自定义的 namespace（命名空间），而不仅仅局限于单个 thread ID。LangGraph 提供了 [stores](persistence.md#memory-store)（[参考文档](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore)）供您保存和回忆 long-term memory。

![](img/memory/short-vs-long.png)


## Short-term memory

[Short-term memory](../how-tos/memory/add-memory.md#add-short-term-memory) 允许您的应用程序在单个 [thread](persistence.md#threads)（会话）或对话中记住之前的交互。一个 [thread](persistence.md#threads) 将一个会话中的多个交互组织起来，类似于电子邮件将同一对话的消息分组的方式。

LangGraph 将 short-term memory 作为 agent 状态的一部分进行管理，并通过 thread 范围内的 checkpoints 来持久化。此状态通常可以包含对话历史以及其他有状态数据，例如上传的文件、检索到的文档或生成的工件。通过将这些信息存储在 graph 的状态中，机器人可以在保持不同 thread 分隔的同时，访问给定对话的完整上下文。

### Manage short-term memory

对话历史是 short-term memory 最常见的形式，而长对话对当前的 LLMs 构成了挑战。完整的历史记录可能无法装入 LLM 的上下文窗口，从而导致不可恢复的错误。即使您的 LLM 支持完整的上下文长度，大多数 LLM 在长上下文上仍然表现不佳。它们会被过时或离题的内容“分散注意力”，同时响应时间变慢且成本更高。

Chat models（聊天模型）使用 messages（消息）来接收上下文，其中包括开发者提供的指令（system message，系统消息）以及用户输入（human messages，用户消息）。在聊天应用程序中，消息在 human input 和 model response 之间交替出现，导致消息列表随时间增长。由于上下文窗口有限，并且包含大量 token 的消息列表可能成本高昂，因此许多应用程序可以从手动删除或忘记过时信息的技术中受益。

![](img/memory/filter.png)

有关管理消息的常见技术的更多信息，请参阅 [Add and manage memory](../how-dos/memory/add-memory.md#manage-short-term-memory) 指南。

## Long-term memory

LangGraph 中的 [Long-term memory](../how-tos/memory/add-memory.md#add-long-term-memory) 允许系统在不同的对话或会话中保留信息。与 short-term memory（它是**thread-scoped**，按会话范围限定）不同，long-term memory 保存在自定义“namespaces”（命名空间）中。

Long-term memory 是一个复杂挑战，没有放之四海而皆准的解决方案。但是，以下问题提供了一个框架，帮助您导航不同的技术：

- [What is the type of memory?](#memory-types)（Memory 的类型是什么？）人类使用 Memory 来记忆事实（[semantic memory](#semantic-memory)，语义记忆）、经验（[episodic memory](#episodic-memory)，情景记忆）和规则（[procedural memory](#procedural-memory)，程序记忆）。AI Agent 也可以以相同的方式使用 Memory。例如，AI Agent 可以使用 Memory 来记住关于用户的特定事实，以便完成任务。

- [When do you want to update memories?](#writing-memories)（何时更新 Memory？）Memory 可以在 agent 的应用程序逻辑中（例如，“in the hot path”，实时路径）进行更新。在这种情况下，agent 通常会在响应用户之前决定要记住哪些事实。或者，Memory 也可以作为后台任务（在后台/异步运行并生成 Memory 的逻辑）进行更新。我们在[下面的部分](#writing-memories)解释了这些方法的权衡。

### Memory types

不同的应用程序需要各种类型的 Memory。尽管类比并不完美，但研究 [human memory types](https://www.psychologytoday.com/us/basics/memory/types-of-memory?ref=blog.langchain.dev)（人类记忆类型）可能很有启发性。一些研究（例如 [CoALA 论文](https://arxiv.org/pdf/2309.02427)）甚至已将这些人类记忆类型映射到 AI Agent 中使用的类型。

| Memory Type | What is Stored | Human Example | Agent Example |
|-------------|----------------|---------------|---------------|
| [Semantic](#semantic-memory) | Facts | Things I learned in school | Facts about a user |
| [Episodic](#episodic-memory) | Experiences | Things I did | Past agent actions |
| [Procedural](#procedural-memory) | Instructions | Instincts or motor skills | Agent system prompt |

#### Semantic memory

[Semantic memory](https://en.wikipedia.org/wiki/Semantic_memory)（语义记忆），无论是人类还是 AI Agent，都涉及对特定事实和概念的保留。对人类而言，它可能包括在学校学到的信息以及对概念及其关系的理解。对于 AI Agent 而言，semantic memory 通常用于通过记住过去交互中的事实或概念来个性化应用程序。

!!! note

    Semantic memory 不同于“semantic search”（语义搜索），后者是一种使用“含义”（通常是 embeddings，嵌入）来查找相似内容的技术。Semantic memory 是一个源自心理学的术语，指的是存储事实和知识，而 semantic search 是一种基于含义而非精确匹配来检索信息的方法。


##### Profile

Semantic memories 可以用不同的方式进行管理。例如，memories 可以是一个单一的、持续更新的“profile”（配置文件），包含有关用户、组织或其他实体（包括 agent 本身）的范围明确且特定的信息。Profile 通常只是一个 JSON 文档，其中包含您选择用于表示域的各种键值对。

在记忆 profile 时，您会希望确保每次都**更新**该 profile。因此，您将需要传入之前的 profile 并[让模型生成新的 profile](https://github.com/langchain-ai/memory-template)（或一些 [JSON patch](https://github.com/hinthornw/trustcall) 应用到旧 profile）。随着 profile 的增大，这可能会变得容易出错，并且可能需要将 profile 分成多个文档或在生成文档时进行**严格**解码，以确保 memory schema 保持有效。

![](img/memory/update-profile.png)

##### Collection

或者，memories 可以是随时间持续更新和扩展的文档集合。每个单独的 memory 可以被更精细地限定范围，并且更容易生成，这意味着您随着时间的推移**丢失**信息的可能性会更小。LLM 更容易为新信息生成_新_对象，而不是协调新信息与现有 profile。因此，文档集合往往能在后续带来[更高的召回率](https://en.wikipedia.org/wiki/Precision_and_recall)。

然而，这会增加 memory 更新的复杂性。模型现在必须_删除_或_更新_列表中的现有项目，这可能很棘手。此外，一些模型可能倾向于过度插入，而另一些模型可能倾向于过度更新。有关管理此问题的一种方法，请参阅 [Trustcall](https://github.com/hinthornw/trustcall) 包，并考虑使用（例如 [LangSmith](https://docs.smith.langchain.com/tutorials/Developers/evaluation) 等工具）进行评估，以帮助您调整行为。

处理文档集合也会将复杂性转移到 memory **搜索**上。`Store` 目前支持[语义搜索](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.query) 和[按内容过滤](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter)。

最后，使用 memory 集合可能会使为模型提供全面的上下文变得具有挑战性。虽然单个 memory 可能遵循特定的 schema，但这可能无法捕获完整的上下文或 memory 之间的关系。因此，当使用这些 memory 生成响应时，模型可能会缺少重要的上下文信息，而这些信息在统一的 profile 方法中可能更容易获得。

![](img/memory/update-list.png)

无论采用何种 memory 管理方法，核心在于 agent 将使用 semantic memories 来[为响应提供依据](https://python.langchain.com/docs/concepts/rag/)，这通常会导致更个性化和相关的交互。

#### Episodic memory

[Episodic memory](https://en.wikipedia.org/wiki/Episodic_memory)（情景记忆），无论是人类还是 AI Agent，都涉及回忆过去的事件或行为。[CoALA 论文](https://arxiv.org/pdf/2309.02427)对此进行了很好的阐述：事实可以写入 semantic memory，而*经验*可以写入 episodic memory。对于 AI Agent 而言，episodic memory 通常用于帮助 agent 记住如何完成任务。

:::python
实际上，episodic memories 通常通过[few-shot 示例提示](https://python.langchain.com/docs/concepts/few_shot_prompting/)来实现，agent 从过去的序列中学习以正确执行任务。有时“展示”比“告知”更容易，LLMs 也很擅长从示例中学习。Few-shot learning 允许您通过更新带有输入-输出示例的提示来“编程”您的 LLM，以说明预期的行为。虽然可以使用各种[最佳实践](https://python.langchain.com/docs/concepts/#1-generating-examples)来生成 few-shot 示例，但挑战通常在于根据用户输入选择最相关的示例。
:::

:::js
实际上，episodic memories 通常通过 few-shot 示例提示来实现，Agent 从过去的序列中学习以正确执行任务。有时“展示”比“告知”更容易，LLMs 也很擅长从示例中学习。Few-shot learning 允许您通过更新带有输入-输出示例的提示来“编程”您的 LLM，以说明预期的行为。虽然可以使用各种最佳实践来生成 few-shot 示例，但挑战通常在于根据用户输入选择最相关的示例。
:::

:::python
请注意，memory [store](persistence.md#memory-store) 只是存储 few-shot 示例数据的一种方式。如果您希望有更多的开发者参与，或者将 few-shot 的应用与评估框架更紧密地结合，您也可以使用 [LangSmith Dataset](https://docs.smith.langchain.com/evaluation/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) 来存储数据。然后，可以现成使用动态 few-shot 示例选择器来实现这一目标。LangSmith 会为您索引数据集，并实现检索与用户输入最相关的 few-shot 示例（基于关键词相似度，[使用类似 BM25 的算法](https://docs.smith.langchain.com/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection)进行关键词相似度匹配）。

请参阅此 [视频](https://www.youtube.com/watch?v=37VaU7e7t5o) 了解 LangSmith 中动态 few-shot 示例选择的示例用法。此外，请参阅这篇[博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/)，展示了如何使用 few-shot 提示来提高工具调用性能，以及这篇[博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/)，展示了如何使用 few-shot 示例来使 LLM 符合人类偏好。
:::

:::js
请注意，memory [store](persistence.md#memory-store) 只是存储 few-shot 示例数据的一种方式。如果您希望有更多的开发者参与，或者将 few-shot 的应用与评估框架更紧密地结合，您也可以使用 LangSmith Dataset 来存储数据。然后，可以现成使用动态 few-shot 示例选择器来实现这一目标。LangSmith 会为您索引数据集，并实现检索与用户输入最相关的 few-shot 示例（基于关键词相似度）。

请参阅此 [视频](https://www.youtube.com/watch?v=37VaU7e7t5o) 了解 LangSmith 中动态 few-shot 示例选择的示例用法。此外，请参阅这篇[博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/)，展示了如何使用 few-shot 提示来提高工具调用性能，以及这篇[博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/)，展示了如何使用 few-shot 示例来使 LLM 符合人类偏好。
:::

#### Procedural memory

[Procedural memory](https://en.wikipedia.org/wiki/Procedural_memory)（程序记忆），无论是在人类还是 AI Agent 中，都涉及到记忆执行任务的规则。对人类而言，procedural memory 就像对如何执行任务的内化知识，例如通过基本的运动技能和平衡来骑自行车。另一方面，episodic memory 涉及回忆具体的经验，例如您第一次不用辅助轮成功骑车，或者一次穿越风景优美路线的难忘骑行。对于 AI Agent 而言，procedural memory 是模型权重、Agent 代码和 Agent 提示的组合，它们共同决定了 Agent 的功能。

实际上，Agent 修改其模型权重或重写其代码的情况相当少见。然而，Agent 修改其自身的提示则更为常见。

一种改进 Agent 指令的有效方法是使用“[]()Reflection”或 meta-prompting（元提示）。这包括使用 Agent 的当前指令（例如，系统提示）以及最近的对话或明确的用户反馈来提示 Agent。然后，Agent 会根据此输入来改进其自身指令。此方法对于指令难以预先确定的任务特别有用，因为它允许 Agent 从其交互中学习和适应。

例如，我们构建了一个[Tweet 生成器](https://www.youtube.com/watch?v=Vn8A3BxfplE)，通过外部反馈和提示重写来为 Twitter 生成高质量的论文摘要。在这种情况下，具体的摘要提示很难事先确定，但用户可以很容易地批判生成的 Tweets 并提供有关如何改进摘要过程的反馈。

下面的伪代码展示了如何使用 LangGraph memory [store](persistence.md#memory-store) 实现这一点，使用 store 来保存提示，`update_instructions` 节点来获取当前提示（以及 `state["messages"]` 中捕获的用户对话反馈），更新提示，并将新提示保存回 store。然后，`call_model` 从 store 中获取更新后的提示并使用它来生成响应。

:::python
```python
# 使用指令的节点
def call_model(state: State, store: BaseStore):
    namespace = ("agent_instructions", )
    instructions = store.get(namespace, key="agent_a")[0]
    # Application logic
    prompt = prompt_template.format(instructions=instructions.value["instructions"])
    ...

# 更新指令的节点
def update_instructions(state: State, store: BaseStore):
    namespace = ("instructions",)
    current_instructions = store.search(namespace)[0]
    # Memory logic
    prompt = prompt_template.format(instructions=current_instructions.value["instructions"], conversation=state["messages"])
    output = llm.invoke(prompt)
    new_instructions = output['new_instructions']
    store.put(("agent_instructions",), "agent_a", {"instructions": new_instructions})
    ...
```
:::

:::js
```typescript
// 使用指令的节点
const callModel = async (state: State, store: BaseStore) => {
    const namespace = ["agent_instructions"];
    const instructions = await store.get(namespace, "agent_a");
    // Application logic
    const prompt = promptTemplate.format({ 
        instructions: instructions[0].value.instructions 
    });
    // ...
};

// 更新指令的节点
const updateInstructions = async (state: State, store: BaseStore) => {
    const namespace = ["instructions"];
    const currentInstructions = await store.search(namespace);
    // Memory logic
    const prompt = promptTemplate.format({ 
        instructions: currentInstructions[0].value.instructions, 
        conversation: state.messages 
    });
    const output = await llm.invoke(prompt);
    const newInstructions = output.new_instructions;
    await store.put(["agent_instructions"], "agent_a", { 
        instructions: newInstructions 
    });
    // ...
};
```
:::

![](img/memory/update-instructions.png)

### Writing memories

Agent 写入 Memory 主要有两种方法：“in the hot path”（实时路径）和“in the background”（后台）。

![](img/memory/hot_path_vs_background.png)

#### In the hot path

在运行时创建 Memory 既有优点也有挑战。好处是，这种方法允许实时更新，使新的 Memory 能够立即用于后续交互。它还增加了透明度，用户可以被告知何时创建和存储了 Memory。

然而，这种方法也带来挑战。如果 agent 需要新工具来决定要将什么内容存入 Memory，这可能会增加复杂性。此外，关于要保存什么到 Memory 的推理过程会影响 Agent 的延迟。最后，Agent 必须在 Memory 创建和其职责之间进行多任务处理，这可能会影响创建 Memory 的数量和质量。

例如，ChatGPT 使用 [save_memories](https://openai.com/index/memory-and-new-controls-for-chatgpt/) 工具将 Memory 作为内容字符串来 upsert（更新或插入），并在每次用户消息时决定是否以及如何使用此工具。请参阅我们的 [memory-agent](https://github.com/langchain-ai/memory-agent) 模板作为参考实现。

#### In the background

将 Memory 作为单独的后台任务创建具有多种优势。它消除了主应用程序的延迟，将应用程序逻辑与 Memory 管理分开，并允许 Agent 进行更集中的任务完成。这种方法还提供了在时间上创建 Memory 的灵活性，以避免冗余工作。

然而，这种方法也有其挑战。确定 Memory 写入的频率至关重要，因为不频繁的更新可能会导致其他 thread 缺少新上下文。决定何时触发 Memory 形成也很重要。常见的策略包括在设定的时间段后安排（如果发生新事件则重新安排），使用 cron 计划，或允许用户或应用程序逻辑手动触发。

请参阅我们的 [memory-service](https://github.com/langchain-ai/memory-template) 模板作为参考实现。

### Memory storage

LangGraph 将 long-term memories 作为 JSON 文档存储在 [store](persistence.md#memory-store)（存储）中。每个 memory 都组织在一个自定义的 `namespace`（命名空间，类似于文件夹）和一个独特的 `key`（键，像文件名）下。Namespaces 通常包含用户或组织 ID 或其他更容易组织信息的标签。这种结构实现了 Memory 的分层组织。跨命名空间的搜索支持通过内容过滤器来实现。

:::python
```python
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # Replace with an actual embedding function or LangChain embeddings object
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore 将数据保存到内存中的字典。在生产环境中使用数据库支持的 store。
store = InMemoryStore(index={"embed": embed, "dims": 2})
user_id = "my-user"
application_context = "chitchat"
namespace = (user_id, application_context)
store.put(
    namespace,
    "a-memory",
    {
        "rules": [
            "User likes short, direct language",
            "User only speaks English & python",
        ],
        "my-key": "my-value",
    },
)
# 通过 ID 获取 "memory"
item = store.get(namespace, "a-memory")
# 在此命名空间中搜索 "memories"，按内容等效性过滤，按向量相似度排序
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)
```
:::

:::js
```typescript
import { InMemoryStore } from "@langchain/langgraph";

const embed = (texts: string[]): number[][] => {
    // Replace with an actual embedding function or LangChain embeddings object
    return texts.map(() => [1.0, 2.0]);
};

// InMemoryStore 将数据保存到内存中的字典。在生产环境中使用数据库支持的 store。
const store = new InMemoryStore({ index: { embed, dims: 2 } });
const userId = "my-user";
const applicationContext = "chitchat";
const namespace = [userId, applicationContext];

await store.put(
    namespace,
    "a-memory",
    {
        rules: [
            "User likes short, direct language",
            "User only speaks English & TypeScript",
        ],
        "my-key": "my-value",
    }
);

// get the "memory" by ID
const item = await store.get(namespace, "a-memory");

// search for "memories" within this namespace, filtering on content equivalence, sorted by vector similarity
const items = await store.search(
    namespace, 
    { 
        filter: { "my-key": "my-value" }, 
        query: "language preferences" 
    }
);
```
:::

有关 memory store 的更多信息，请参阅 [Persistence](persistence.md#memory-store) 指南。