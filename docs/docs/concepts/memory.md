---
search:
  boost: 2
---

# Memory

[Memory](../how-tos/memory/add-memory.md) 是一个用于记住先前交互信息的系统。对 AI 代理而言，记忆至关重要，因为它能让代理记住先前的交互、从反馈中学习并适应用户偏好。随着代理承担越来越复杂的任务以及交互次数的增加，这种能力对于效率和用户满意度都至关重要。

本概念指南涵盖了两种类型的内存，基于它们的召回范围：

- [短期记忆](#short-term-memory)，或称 [线程](persistence.md#threads) 范围的内存，通过在会话中维护消息历史来跟踪当前对话。LangGraph 将短期记忆作为代理 [状态](low_level.md#state) 的一部分来管理。状态使用 [checkpointer](persistence.md#checkpoints) 持久化到数据库中，以便随时恢复线程。短期记忆在图被调用或步骤完成时更新，并且状态在每个步骤开始时读取。

- [长期记忆](#long-term-memory) 在会话之间存储用户特定或应用程序级别的数据，并在不同的对话线程之间共享。它可以 _随时_ 和 _在任何线程中_ 被召回。内存的范围限于任何自定义命名空间，而不仅仅是单个线程 ID 内。LangGraph 提供 [存储](persistence.md#memory-store)（[参考文档](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.BaseStore)）以允许您保存和召回长期记忆。

![](img/memory/short-vs-long.png)


## 短期记忆

[短期记忆](../how-tos/memory/add-memory.md#add-short-term-memory) 让您的应用程序能够记住单个 [线程](persistence.md#threads) 或对话中的先前交互。一个 [线程](persistence.md#threads) 会将一个会话中的多次交互组织起来，类似于电子邮件将消息分组到单个对话中的方式。

LangGraph 将短期记忆作为代理状态的一部分进行管理，该状态通过线程范围的检查点进行持久化。此状态通常可以包含对话历史以及其他有状态数据，例如上传的文件、检索到的文档或生成的工件。通过将这些数据存储在图的状态中，机器人可以访问给定对话的完整上下文，同时维护不同线程之间的隔离。

### 管理短期记忆

对话历史是最常见的短期记忆形式，而长对话给当今的 LLM 带来了挑战。完整的历史可能无法放入 LLM 的上下文窗口中，从而导致无法恢复的错误。即使您的 LLM 支持完整的上下文长度，大多数 LLM 在长上下文中的表现仍然很差。它们会被陈旧或离题的内容“分心”，同时响应时间变慢且成本更高。

聊天模型使用消息来接受上下文，这些消息包括开发人员提供的指令（系统消息）和用户输入（人类消息）。在聊天应用程序中，消息在人类输入和模型响应之间交替，从而产生一个随时间增长的消息列表。由于上下文窗口有限且包含大量标记的消息列表可能成本高昂，因此许多应用程序可以受益于使用手动删除或遗忘过时信息的技术。

![](img/memory/filter.png)

有关管理消息的常见技术的更多信息，请参阅 [添加和管理内存](../how-tos/memory/add-memory.md#manage-short-term-memory) 指南。

## 长期记忆

LangGraph 中的 [长期记忆](../how-tos/memory/add-memory.md#add-long-term-memory) 允许系统在不同的对话或会话中保留信息。与仅限于线程范围的短期记忆不同，长期记忆保存在自定义“命名空间”中。

长期记忆是一个复杂的挑战，没有一刀切的解决方案。但是，以下问题提供了一个框架，可帮助您了解不同的技术：

- [内存的类型是什么？](#memory-types) 人类使用记忆来记住事实（[语义记忆](#semantic-memory)）、经历（[情景记忆](#episodic-memory)）和规则（[程序记忆](#procedural-memory)）。AI 代理也可以以同样的方式使用记忆。例如，AI 代理可以使用记忆来记住有关用户的特定事实以完成任务。

- [您想何时更新记忆？](#writing-memories) 记忆可以在代理的应用程序逻辑中（例如，“在热路径中”）进行更新。在这种情况下，代理通常会在回复用户之前决定要记住的事实。或者，记忆可以作为后台任务进行更新（在后台/异步运行并生成记忆的逻辑）。我们在 [下面的部分](#writing-memories) 中解释了这些方法的权衡。

### 内存类型

不同的应用程序需要各种类型的内存。尽管类比并不完美，但研究 [人类记忆类型](https://www.psychologytoday.com/us/basics/memory/types-of-memory?ref=blog.langchain.dev) 可能会有所启发。一些研究（例如 [CoALA 论文](https://arxiv.org/pdf/2309.02427)）甚至将这些人类记忆类型映射到 AI 代理使用的记忆类型中。

| 内存类型 | 存储内容 | 人类示例 | 代理示例 |
|---|---|---|---|
| [语义](#semantic-memory) | 事实 | 我在学校学到的东西 | 关于用户的知识 |
| [情景](#episodic-memory) | 经历 | 我做过的事情 | 先前代理的操作 |
| [程序](#procedural-memory) | 指令 | 直觉或运动技能 | 代理系统提示 |

#### 语义记忆

[语义记忆](https://en.wikipedia.org/wiki/Semantic_memory) 在人类和 AI 代理中都涉及对特定事实和概念的保留。在人类中，它包括从学校学到的信息以及对概念及其关系的理解。对于 AI 代理，语义记忆通常通过记住过去的交互中的事实或概念来个性化应用程序。

!!! note

    语义记忆不同于“语义搜索”，语义搜索是一种使用“含义”（通常是嵌入）来查找相似内容的技术。语义记忆是心理学中的一个术语，指存储事实和知识，而语义搜索是一种根据含义而非精确匹配来检索信息的方法。


##### 个人资料

语义记忆可以通过不同的方式进行管理。例如，记忆可以是一个单一的、持续更新的特定领域（包括代理本身）的用户、组织或其他实体的信息“个人资料”。个人资料通常只是一个 JSON 文档，其中包含您为表示您的域而选择的各种键值对。

在记忆个人资料时，您需要确保每次都在 [更新](https://github.com/langchain-ai/memory-template) 它。因此，您将需要传入之前的个人资料并 [要求模型生成新的个人资料](https://github.com/langchain-ai/memory-template)（或应用到旧个人资料的 [JSON patch](https://github.com/hinthornw/trustcall)）。随着个人资料的增大，这可能会导致错误，并且可能需要将个人资料拆分成多个文档或在生成文档时使用 [严格](https://github.com/hinthornw/trustcall) 解码，以确保内存模式保持有效。

![](img/memory/update-profile.png)

##### 集合

或者，记忆可以是在一段时间内不断更新和扩展的文档集合。每个单独的记忆可以具有更狭窄的范围，并且更容易生成，这意味着您不太可能随着时间的推移而 [丢失信息](https://en.wikipedia.org/wiki/Precision_and_recall)。LLM 更容易为新信息生成 _新_ 对象，而不是将新信息与现有个人资料进行协调。因此，文档集合倾向于 [提高下游召回率](https://en.wikipedia.org/wiki/Precision_and_recall)。

但是，这会将一些复杂性转移到内存更新中。现在模型必须 _删除_ 或 _更新_ 列表中的现有项，这可能会很棘手。此外，一些模型可能默认为过度插入，而另一些则可能默认为过度更新。请参阅 [Trustcall](https://github.com/hinthornw/trustcall) 包来管理此问题的一种方法，并考虑使用 [LangSmith](https://docs.smith.langchain.com/tutorials/Developers/evaluation) 等工具进行评估，以帮助您调整行为。

使用文档集合也使内存搜索 [列表](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter) 的复杂性转移到内存搜索上。`Store` 目前支持 [语义搜索](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.query) 和 [按内容过滤](https://langchain-ai.github.io/langgraph/reference/store/#langgraph.store.base.SearchOp.filter)。

最后，使用内存集合可能会使为模型提供全面上下文变得困难。虽然单个记忆可能遵循特定模式，但这种结构可能无法捕获记忆之间的完整上下文或关系。因此，在使用这些记忆生成响应时，模型可能会缺少一些重要上下文信息，而这些信息在统一的个人资料方法中会更加容易获得。

![](img/memory/update-list.png)

无论采用何种内存管理方法，核心在于代理将使用语义记忆来 [固定其响应](https://python.langchain.com/docs/concepts/rag/)，这通常会导致更具个性化和更相关的交互。

#### 情景记忆

[情景记忆](https://en.wikipedia.org/wiki/Episodic_memory) 在人类和 AI 代理中都涉及回忆过去的事件或操作。[CoALA 论文](https://arxiv.org/pdf/2309.02427) 对此进行了很好的阐述：事实可以写入语义记忆，而 *经历* 可以写入情景记忆。对于 AI 代理，情景记忆通常用于帮助代理记住如何完成任务。

在实践中，情景记忆通常通过 [少样本示例提示](https://python.langchain.com/docs/concepts/few_shot_prompting/) 来实现，代理从中学习过去的序列以正确执行任务。有时“展示”比“告知”更容易，LLM 可以从示例中很好地学习。少样本学习允许您通过使用输入-输出示例更新提示来“编程”您的 LLM，以说明预期的行为。虽然可以使用各种 [最佳实践](https://python.langchain.com/docs/concepts/#1-generating-examples) 来生成少样本示例，但挑战通常在于根据用户输入选择最相关的示例。

请注意，内存 [存储](persistence.md#memory-store) 只是存储数据作为少样本示例的一种方式。如果您希望有更多的开发者参与，或者将少样本示例与您的评估套件更紧密地结合，您还可以使用 [LangSmith 数据集](https://docs.smith.langchain.com/evaluation/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) 来存储您的数据。然后，可以使用开箱即用的动态少样本示例选择器来实现相同的目标。LangSmith 将为您索引数据集，并允许您根据关键字相似性检索与用户输入最相关的少样本示例（[使用类似 BM25 的算法](https://docs.smith.langchain.com/how_to_guides/datasets/index_datasets_for_dynamic_few_shot_example_selection) 来实现基于关键字的相似性）。

有关在 LangSmith 中使用动态少样本示例选择的示例用法，请参阅此如何操作 [视频](https://www.youtube.com/watch?v=37VaU7e7t5o)。另请参阅此 [博客文章](https://blog.langchain.dev/few-shot-prompting-to-improve-tool-calling-performance/)，展示如何使用少样本提示来提高工具调用性能，以及此 [博客文章](https://blog.langchain.dev/aligning-llm-as-a-judge-with-human-preferences/)，展示如何使用少样本示例来使 LLM 与人类偏好保持一致。

#### 程序记忆

[程序记忆](https://en.wikipedia.org/wiki/Procedural_memory) 在人类和 AI 代理中都涉及对执行任务的规则的记忆。在人类中，程序记忆就像对如何执行任务的内在知识，例如通过基本的运动技能和平衡来骑自行车。另一方面，情景记忆涉及回忆具体的经历，例如第一次成功地在没有辅助轮的情况下骑自行车，或者一次穿越风景优美的路线的难忘骑行。对于 AI 代理，程序记忆是模型权重、代理代码和代理提示的组合，它们共同决定了代理的功能。

在实践中，代理修改其模型权重或重写其代码是相当不常见的。然而，代理修改其本身的提示更为常见。

改进代理指令的一种有效方法是通过“ [反思](https://blog.langchain.com/reflection-agents/)”或元提示。这包括使用代理当前的指令（例如，系统提示）、最近的对话或明确的用户反馈来提示代理。然后，代理根据此输入来优化其自身的指令。此方法对于指令难以预先指定的任务特别有用，因为它允许代理从交互中学习和适应。

例如，我们使用外部反馈和提示重写构建了一个 [Tweet 生成器](https://www.youtube.com/watch?v=Vn8A3BxfplE)，以生成高质量的推特论文摘要。在这种情况下，具体的摘要提示很难事先指定*，但用户很容易批评生成的推文并提供有关如何改进摘要过程的反馈。

下面的伪代码展示了如何使用 LangGraph 内存 [存储](persistence.md#memory-store) 来实现此功能，使用存储来保存提示，使用 `update_instructions` 节点来获取当前提示（以及用户在 `state["messages"]` 中捕获的对话反馈），更新提示，并将新提示保存回存储。然后，`call_model` 从存储中获取更新的提示并使用它来生成响应。

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
    prompt = prompt_template.format(instructions=instructions.value["instructions"], conversation=state["messages"])
    output = llm.invoke(prompt)
    new_instructions = output['new_instructions']
    store.put(("agent_instructions",), "agent_a", {"instructions": new_instructions})
    ...
```

![](img/memory/update-instructions.png)

### 写入记忆

代理写入记忆主要有两种方法：“ [在热路径中](#in-the-hot-path)”和“ [在后台](#in-the-background)”。

![](img/memory/hot_path_vs_background.png)

#### 在热路径中

在运行时创建记忆既有优点也有挑战。从积极的方面来看，这种方法允许实时更新，使新记忆可供后续交互立即使用。它还提供了透明性，因为用户可以在创建和存储记忆时收到通知。

但是，此方法也带来了挑战。如果代理需要新工具来决定要将什么提交到内存，这可能会增加复杂性。此外，关于要保存在内存中的内容的推理过程可能会影响代理的延迟。最后，代理必须在记忆创建和其他职责之间进行多任务处理，这可能会影响创建的记忆的数量和质量。

例如，ChatGPT 使用 [save_memories](https://openai.com/index/memory-and-new-controls-for-chatgpt/) 工具将内容字符串保存到内存中，并决定每次用户消息是否以及如何使用此工具。请参阅我们的 [memory-agent](https://github.com/langchain-ai/memory-agent) 模板作为参考实现。

#### 在后台

将记忆作为单独的后台任务创建具有多种优势。它消除了主要应用程序的延迟，将应用程序逻辑与内存管理分开，并允许代理更专注地完成任务。这种方法还提供了在内存创建时间选择上的灵活性，以避免重复工作。

但是，此方法本身也存在挑战。确定内存写入的频率变得至关重要，因为不频繁的更新可能会使其他线程缺少新上下文。决定何时触发内存形成也很重要。常见策略包括在设定的时间段后安排（如有新事件发生则重新安排）、使用 cron 计划或允许用户或应用程序逻辑进行手动触发。

请参阅我们的 [memory-service](https://github.com/langchain-ai/memory-template) 模板作为参考实现。

### 内存存储

LangGraph 将长期记忆存储为 [存储](persistence.md#memory-store) 中的 JSON 文档。每个内存都组织在一个自定义的 `namespace`（类似于文件夹）和一个独特的 `key`（类似于文件名）下。命名空间通常包含用户或组织 ID 或其他使信息更易于组织的标签。这种结构使得内存的层级化组织成为可能。然后，通过内容过滤器支持跨命名空间的搜索。

```python
from langgraph.store.memory import InMemoryStore


def embed(texts: list[str]) -> list[list[float]]:
    # Replace with an actual embedding function or LangChain embeddings object
    return [[1.0, 2.0] * len(texts)]


# InMemoryStore 将数据保存到内存中的字典中。在生产环境中使用基于数据库的存储。
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
# 在此命名空间内搜索 "memories"，按内容等价性过滤，按向量相似性排序
items = store.search(
    namespace, filter={"my-key": "my-value"}, query="language preferences"
)
```

有关内存存储的更多信息，请参阅 [持久化](persistence.md#memory-store) 指南。