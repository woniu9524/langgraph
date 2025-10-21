---
search:
  boost: 2
tags:
  - human-in-the-loop
  - hil
  - interrupt
hide:
  - tags
---

# 启用人工干预

要审核、编辑和批准代理或工作流中的工具调用，请使用中断（interrupts）来暂停图（graph）并等待人工输入。中断利用 LangGraph 的 [持久化](../../concepts/persistence.md) 层来保存图状态，从而无限期地暂停图执行，直到您恢复它。

!!! info

    有关人工干预工作流的更多信息，请参阅[人工干预](../../concepts/human_in_the_loop.md)概念指南。

## 使用 `interrupt` 暂停

:::python
[动态中断](../../concepts/human_in_the_loop.md#key-capabilities)（也称为动态断点）会根据图的当前状态触发。您可以通过在适当的位置调用 @[`interrupt` 函数][interrupt] 来设置动态中断。图将暂停，允许人工干预，然后根据其输入恢复图。这对于审批、编辑或收集额外上下文等任务非常有用。

!!! note

    截至 v1.0，`interrupt` 是暂停图的推荐方式。`NodeInterrupt` 已弃用，并将在 v2.0 中移除。

:::

:::js
[动态中断](../../concepts/human_in_the_loop.md#key-capabilities)（也称为动态断点）会根据图的当前状态触发。您可以通过在适当的位置调用 @[`interrupt` 函数][interrupt] 来设置动态中断。图将暂停，允许人工干预，然后根据其输入恢复图。这对于审批、编辑或收集额外上下文等任务非常有用。
:::

要在图中启用 `interrupt`，您需要：

1. [**指定一个检查点（checkpointer）**](../../concepts/persistence.md#checkpoints) 以在每一步后保存图状态。
2. **调用 `interrupt()`** 在适当位置。有关示例，请参阅[通用模式](#common-patterns)部分。
3. **使用 [**线程 ID**](../../concepts/persistence.md#threads) 运行图**，直到命中 `interrupt`。
4. **使用 `invoke`/`stream` 恢复执行**（请参阅[**`Command` 原语**](#resume-using-the-command-primitive)）。

:::python

```python
# highlight-next-line
from langgraph.types import interrupt, Command

def human_node(state: State):
    # highlight-next-line
    value = interrupt( # (1)!
        {
            "text_to_revise": state["some_text"] # (2)!
        }
    )
    return {
        "some_text": value # (3)!
    }


graph = graph_builder.compile(checkpointer=checkpointer) # (4)!

# 运行图直到命中中断。
config = {"configurable": {"thread_id": "some_id"}}
result = graph.invoke({"some_text": "original text"}, config=config) # (5)!
print(result['__interrupt__']) # (6)!
# > [
# >    Interrupt(
# >       value={'text_to_revise': 'original text'},
# >       resumable=True,
# >       ns=['human_node:6ce9e64f-edef-fe5d-f7dc-511fa9526960']
# >    )
# > ]

# highlight-next-line
print(graph.invoke(Command(resume="Edited text"), config=config)) # (7)!
# > {'some_text': 'Edited text'}
```

1. `interrupt(...)` 在 `human_node` 处暂停执行，将给定有效负载（payload）显示给人类。
2. 任何 JSON 可序列化值都可以传递给 `interrupt` 函数。此处，传递了一个包含待修订文本的字典。
3. 恢复后，`interrupt(...)` 的返回值是用户提供的输入，用于更新状态。
4. 需要检查点（checkpointer）来持久化图状态。在生产环境中，这应该是持久的（例如，由数据库支持）。
5. 图以一些初始状态调用（invoke）。
6. 当图命中中断时，它会返回一个包含有效负载和元数据的 `Interrupt` 对象。
7. 图使用 `Command(resume=...)` 恢复，注入人类的输入并继续执行。
   :::

:::js

```typescript
// highlight-next-line
import { interrupt, Command } from "@langchain/langgraph";

const graph = graphBuilder
  .addNode("humanNode", (state) => {
    // highlight-next-line
    const value = interrupt( // (1)!
      // (2)!
      {
        textToRevise: state.someText,
      }
    );
    return {
      someText: value, // (3)!
    };
  })
  .addEdge(START, "humanNode")
  .compile({ checkpointer }); // (4)!

// 运行图直到命中中断。
const config = { configurable: { thread_id: "some_id" } };
const result = await graph.invoke({ someText: "original text" }, config); // (5)!
console.log(result.__interrupt__); // (6)!
// > [
// >   {
// >     value: { textToRevise: 'original text' },
// >     resumable: true,
// >     ns: ['humanNode:6ce9e64f-edef-fe5d-f7dc-511fa9526960'],
// >     when: 'during'
// >   }
// > ]

// highlight-next-line
console.log(await graph.invoke(new Command({ resume: "Edited text" }), config)); // (7)!
// > { someText: 'Edited text' }
```

1. `interrupt(...)` 在 `humanNode` 处暂停执行，将给定有效负载（payload）显示给人类。
2. 任何 JSON 可序列化值都可以传递给 `interrupt` 函数。此处，传递了一个包含待修订文本的对象。
3. 恢复后，`interrupt(...)` 的返回值是用户提供的输入，用于更新状态。
4. 需要检查点（checkpointer）来持久化图状态。在生产环境中，这应该是持久的（例如，由数据库支持）。
5. 图以一些初始状态调用（invoke）。
6. 当图命中中断时，它会返回一个包含 `__interrupt__` 的对象，其中包含有效负载和元数据。
7. 图使用 `Command({ resume: ... })` 恢复，注入人类的输入并继续执行。
   :::

??? example "Extended example: using `interrupt`"

    :::python
    ```python
    from typing import TypedDict
    import uuid
    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.constants import START
    from langgraph.graph import StateGraph

    # highlight-next-line
    from langgraph.types import interrupt, Command


    class State(TypedDict):
        some_text: str


    def human_node(state: State):
        # highlight-next-line
        value = interrupt( # (1)!
            {
                "text_to_revise": state["some_text"] # (2)!
            }
        )
        return {
            "some_text": value # (3)!
        }


    # Build the graph
    graph_builder = StateGraph(State)
    graph_builder.add_node("human_node", human_node)
    graph_builder.add_edge(START, "human_node")
    checkpointer = InMemorySaver() # (4)!
    graph = graph_builder.compile(checkpointer=checkpointer)
    # Pass a thread ID to the graph to run it.
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    # Run the graph until the interrupt is hit.
    result = graph.invoke({"some_text": "original text"}, config=config) # (5)!

    print(result['__interrupt__']) # (6)!
    # > [
    # >    Interrupt(
    # >       value={'text_to_revise': 'original text'},
    # >       resumable=True,
    # >       ns=['human_node:6ce9e64f-edef-fe5d-f7dc-511fa9526960']
    # >    )
    # > ]
    print(result["__interrupt__"]) # (6)!
    # > [Interrupt(value={'text_to_revise': 'original text'}, id='6d7c4048049254c83195429a3659661d')]

    # highlight-next-line
    print(graph.invoke(Command(resume="Edited text"), config=config)) # (7)!
    # > {'some_text': 'Edited text'}
    ```

    1. `interrupt(...)` pauses execution at `human_node`, surfacing the given payload to a human.
    2. Any JSON serializable value can be passed to the `interrupt` function. Here, a dict containing the text to revise.
    3. Once resumed, the return value of `interrupt(...)` is the human-provided input, which is used to update the state.
    4. A checkpointer is required to persist graph state. In production, this should be durable (e.g., backed by a database).
    5. The graph is invoked with some initial state.
    6. When the graph hits the interrupt, it returns an `Interrupt` object with the payload and metadata.
    7. The graph is resumed with a `Command(resume=...)`, injecting the human's input and continuing execution.
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import { MemorySaver, StateGraph, START, interrupt, Command } from "@langchain/langgraph";

    const StateAnnotation = z.object({
      someText: z.string(),
    });

    // Build the graph
    const graphBuilder = new StateGraph(StateAnnotation)
      .addNode("humanNode", (state) => {
        // highlight-next-line
        const value = interrupt( // (1)!
          // (2)!
          {
            textToRevise: state.someText
          }
        );
        return {
          someText: value // (3)!
        };
      })
      .addEdge(START, "humanNode");

    const checkpointer = new MemorySaver(); // (4)!

    const graph = graphBuilder.compile({ checkpointer });

    // Pass a thread ID to the graph to run it.
    const config = { configurable: { thread_id: uuidv4() } };

    // Run the graph until the interrupt is hit.
    const result = await graph.invoke({ someText: "original text" }, config); // (5)!

    console.log(result.__interrupt__); // (6)!
    // > [
    // >   {
    // >     value: { textToRevise: 'original text' },
    // >     resumable: true,
    // >     ns: ['humanNode:6ce9e64f-edef-fe5d-f7dc-511fa9526960'],
    // >     when: 'during'
    // >   }
    // > ]

    // highlight-next-line
    console.log(await graph.invoke(new Command({ resume: "Edited text" }), config)); // (7)!
    // > { someText: 'Edited text' }
    ```

    1. `interrupt(...)` pauses execution at `humanNode`, surfacing the given payload to a human.
    2. Any JSON serializable value can be passed to the `interrupt` function. Here, an object containing the text to revise.
    3. Once resumed, the `interrupt(...)` 的返回值是用户提供的输入，用于更新状态。
    4. A checkpointer is required to persist graph state. In production, this should be durable (e.g., backed by a database).
    5. The graph is invoked with some initial state.
    6. When the graph hits the interrupt, it returns an object with `__interrupt__` containing the payload and metadata.
    7. The graph is resumed with a `Command({ resume: ... })`, injecting the human's input and continuing execution.
    :::

!!! tip "New in 0.4.0"

    :::python
    `__interrupt__` 是一个特殊键，当图被中断时运行图会返回该键。`invoke` 和 `ainvoke` 对 `__interrupt__` 的支持已在 0.4.0 版本中添加。如果您使用的是旧版本，只有在使用 `stream` 或 `astream` 时才能看到 `__interrupt__`。您还可以使用 `graph.get_state(thread_id)` 来获取中断值。
    :::

    :::js
    `__interrupt__` 是图中中断时返回的特殊键。`invoke` 对 `__interrupt__` 的支持已在 0.4.0 版本中添加。如果您使用的是旧版本，只有在使用 `stream` 时才能看到 `__interrupt__`。您还可以使用 `graph.getState(config)` 来获取中断值。
    :::

!!! warning

    :::python
    中断在开发者体验方面类似于 Python 的 `input()` 函数，但它们不会自动从中断点恢复执行。相反，它们会重新运行使用中断的整个节点。因此，中断通常最好放在节点的开头或一个单独的节点中。
    :::

    :::js
    中断既强大又便捷，但需要注意的是，它们不会自动从中断点恢复执行。相反，它们会重新运行使用中断的整个节点。因此，中断通常最好放在节点的开头或一个单独的节点中。
    :::

## 使用 `Command` 原语恢复

:::python
!!! warning

    从 `interrupt` 恢复与 Python 的 `input()` 函数不同，后者是从调用 `input()` 函数的确切点恢复执行。

:::

当 `interrupt` 函数在图中被使用时，执行会在该点暂停，并等待用户输入。

:::python
要恢复执行，请使用 @[`Command`][Command] 原语，该原语可以通过 `invoke` 或 `stream` 方法提供。图将从调用 `interrupt(...)` 的节点的开头恢复执行。这一次，`interrupt` 函数将返回 `Command(resume=value)` 中提供的值，而不是再次暂停。从节点开头到 `interrupt` 的所有代码都将被重新执行。

```python
# 通过提供用户的输入来恢复图执行。
graph.invoke(Command(resume={"age": "25"}), thread_config)
```

:::

:::js
要恢复执行，请使用 @[`Command`][Command] 原语，该原语可以通过 `invoke` 或 `stream` 方法提供。图将从调用 `interrupt(...)` 的节点的开头恢复执行。这一次，`interrupt` 函数将返回 `Command(resume=value)` 中提供的值，而不是再次暂停。从节点开头到 `interrupt` 的所有代码都将被重新执行。

```typescript
// 通过提供用户的输入来恢复图执行。
await graph.invoke(new Command({ resume: { age: "25" } }), threadConfig);
```

:::

### 使用一次调用恢复多个中断

当节点以并行方式运行中断条件时，任务队列中可能存在多个中断。
例如，以下图并行运行两个需要人工输入的节点：

<figure markdown="1">
![image](../assets/human_in_loop_parallel.png){: style="max-height:400px"}
</figure>

:::python
一旦您的图被中断并停滞，您就可以使用 `Command.resume` 一次性恢复所有中断，提供一个中断 ID 到恢复值的映射。

```python
from typing import TypedDict
import uuid
from langchain_core.runnables import RunnableConfig
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.constants import START
from langgraph.graph import StateGraph
from langgraph.types import interrupt, Command


class State(TypedDict):
    text_1: str
    text_2: str


def human_node_1(state: State):
    value = interrupt({"text_to_revise": state["text_1"]})
    return {"text_1": value}


def human_node_2(state: State):
    value = interrupt({"text_to_revise": state["text_2"]})
    return {"text_2": value}


graph_builder = StateGraph(State)
graph_builder.add_node("human_node_1", human_node_1)
graph_builder.add_node("human_node_2", human_node_2)

# 从 START 并行添加两个节点
graph_builder.add_edge(START, "human_node_1")
graph_builder.add_edge(START, "human_node_2")

checkpointer = InMemorySaver()
graph = graph_builder.compile(checkpointer=checkpointer)

thread_id = str(uuid.uuid4())
config: RunnableConfig = {"configurable": {"thread_id": thread_id}}
result = graph.invoke(
    {"text_1": "original text 1", "text_2": "original text 2"}, config=config
)

# 使用中断 ID 到值的映射进行恢复
resume_map = {
    i.id: f"edited text for {i.value['text_to_revise']}"
    for i in graph.get_state(config).interrupts
}
print(graph.invoke(Command(resume=resume_map), config=config))
# > {'text_1': 'edited text for original text 1', 'text_2': 'edited text for original text 2'}
```

:::

:::js

```typescript
const state = await parentGraph.getState(threadConfig);
const resumeMap = Object.fromEntries(
  state.interrupts.map((i) => [
    i.interruptId,
    `human input for prompt ${i.value}`,
  ])
);

await parentGraph.invoke(new Command({ resume: resumeMap }), threadConfig);
```

:::

## 通用模式

下面我们展示了可以使用 `interrupt` 和 `Command` 实现的不同设计模式。

### 批准或拒绝

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/approve-or-reject.png){: style="max-height:400px"}
<figcaption>根据人类的批准或拒绝，图可以继续执行操作或采取替代路径。</figcaption>
</figure>

在关键步骤（如 API 调用）之前暂停图，以审核和批准操作。如果操作被拒绝，您可以阻止图执行该步骤，并可能采取替代操作。

:::python

```python
from typing import Literal
from langgraph.types import interrupt, Command

def human_approval(state: State) -> Command[Literal["some_node", "another_node"]]:
    is_approved = interrupt(
        {
            "question": "Is this correct?",
            # 显示应由人类审核和批准的输出。
            "llm_output": state["llm_output"]
        }
    )

    if is_approved:
        return Command(goto="some_node")
    else:
        return Command(goto="another_node")

# 在适当的位置将节点添加到图中
# 并将其连接到相关节点。
graph_builder.add_node("human_approval", human_approval)
graph = graph_builder.compile(checkpointer=checkpointer)

# 运行图并命中中断后，图将暂停。
# 使用批准或拒绝来恢复它。
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(Command(resume=True), config=thread_config)
```

:::

:::js

```typescript
import { interrupt, Command } from "@langchain/langgraph";

// 在适当的位置将节点添加到图中
// 并将其连接到相关节点。
graphBuilder.addNode("humanApproval", (state) => {
  const isApproved = interrupt({
    question: "Is this correct?",
    // 显示应由人类审核和批准的输出。
    llmOutput: state.llmOutput,
  });

  if (isApproved) {
    return new Command({ goto: "someNode" });
  } else {
    return new Command({ goto: "anotherNode" });
  }
});
const graph = graphBuilder.compile({ checkpointer });

// 运行图并命中中断后，图将暂停。
// 使用批准或拒绝来恢复它。
const threadConfig = { configurable: { thread_id: "some_id" } };
await graph.invoke(new Command({ resume: true }), threadConfig);
```

:::

??? example "Extended example: approve or reject with interrupt"

    :::python
    ```python
    from typing import Literal, TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define the shared graph state
    class State(TypedDict):
        llm_output: str
        decision: str

    # Simulate an LLM output node
    def generate_llm_output(state: State) -> State:
        return {"llm_output": "This is the generated output."}

    # Human approval node
    def human_approval(state: State) -> Command[Literal["approved_path", "rejected_path"]]:
        decision = interrupt({
            "question": "Do you approve the following output?",
            "llm_output": state["llm_output"]
        })

        if decision == "approve":
            return Command(goto="approved_path", update={"decision": "approved"})
        else:
            return Command(goto="rejected_path", update={"decision": "rejected"})

    # Next steps after approval
    def approved_node(state: State) -> State:
        print("✅ Approved path taken.")
        return state

    # Alternative path after rejection
    def rejected_node(state: State) -> State:
        print("❌ Rejected path taken.")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("generate_llm_output", generate_llm_output)
    builder.add_node("human_approval", human_approval)
    builder.add_node("approved_path", approved_node)
    builder.add_node("rejected_path", rejected_node)

    builder.set_entry_point("generate_llm_output")
    builder.add_edge("generate_llm_output", "human_approval")
    builder.add_edge("approved_path", END)
    builder.add_edge("rejected_path", END)

    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Run until interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)
    print(result["__interrupt__"])
    # Output:
    # Interrupt(value={'question': 'Do you approve the following output?', 'llm_output': 'This is the generated output.'}, ...)

    # Simulate resuming with human input
    # To test rejection, replace resume="approve" with resume="reject"
    final_result = graph.invoke(Command(resume="approve"), config=config)
    print(final_result)
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define the shared graph state
    const StateAnnotation = z.object({
      llmOutput: z.string(),
      decision: z.string(),
    });

    // Simulate an LLM output node
    function generateLlmOutput(state: z.infer<typeof StateAnnotation>) {
      return { llmOutput: "This is the generated output." };
    }

    // Human approval node
    function humanApproval(state: z.infer<typeof StateAnnotation>): Command {
      const decision = interrupt({
        question: "Do you approve the following output?",
        llmOutput: state.llmOutput
      });

      if (decision === "approve") {
        return new Command({
          goto: "approvedPath",
          update: { decision: "approved" }
        });
      } else {
        return new Command({
          goto: "rejectedPath",
          update: { decision: "rejected" }
        });
      }
    }

    // Next steps after approval
    function approvedNode(state: z.infer<typeof StateAnnotation>) {
      console.log("✅ Approved path taken.");
      return state;
    }

    // Alternative path after rejection
    function rejectedNode(state: z.infer<typeof StateAnnotation>) {
      console.log("❌ Rejected path taken.");
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("generateLlmOutput", generateLlmOutput)
      .addNode("humanApproval", humanApproval, {
        ends: ["approvedPath", "rejectedPath"]
      })
      .addNode("approvedPath", approvedNode)
      .addNode("rejectedPath", rejectedNode)
      .addEdge(START, "generateLlmOutput")
      .addEdge("generateLlmOutput", "humanApproval")
      .addEdge("approvedPath", END)
      .addEdge("rejectedPath", END);

    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Run until interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await graph.invoke({}, config);
    console.log(result.__interrupt__);
    // Output:
    // [{
    //   value: {
    //     question: 'Do you approve the following output?',
    //     llmOutput: 'This is the generated output.'
    //   },
    //   ...
    // }]

    // Simulate resuming with human input
    // To test rejection, replace resume: "approve" with resume: "reject"
    const finalResult = await graph.invoke(
      new Command({ resume: "approve" }),
      config
    );
    console.log(finalResult);
    ```
    :::

### 审核和编辑状态

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/edit-graph-state-simple.png){: style="max-height:400px"}
<figcaption>人类可以审核和编辑图的状态。这对于纠正错误或使用额外信息更新状态非常有用。
</figcaption>
</figure>

:::python

```python
from langgraph.types import interrupt

def human_editing(state: State):
    ...
    result = interrupt(
        # 向客户端显示的中断信息。
        # 任何 JSON 可序列化值都可以。
        {
            "task": "Review the output from the LLM and make any necessary edits.",
            "llm_generated_summary": state["llm_generated_summary"]
        }
    )

    # 使用编辑后的文本更新状态
    return {
        "llm_generated_summary": result["edited_text"]
    }

# 在适当的位置将节点添加到图中
# 并将其连接到相关节点。
graph_builder.add_node("human_editing", human_editing)
graph = graph_builder.compile(checkpointer=checkpointer)

...

# 运行图并命中中断后，图将暂停。
# 使用编辑后的文本恢复它。
thread_config = {"configurable": {"thread_id": "some_id"}}
graph.invoke(
    Command(resume={"edited_text": "The edited text"}),
    config=thread_config
)
```

:::

:::js

```typescript
import { interrupt } from "@langchain/langgraph";

function humanEditing(state: z.infer<typeof StateAnnotation>) {
  const result = interrupt({
    // 向客户端显示的中断信息。
    # 任何 JSON 可序列化值都可以。
    task: "Review the output from the LLM and make any necessary edits.",
    llmGeneratedSummary: state.llmGeneratedSummary,
  });

  // 使用编辑后的文本更新状态
  return {
    llmGeneratedSummary: result.editedText,
  };
}

// 在适当的位置将节点添加到图中
// 并将其连接到相关节点。
graphBuilder.addNode("humanEditing", humanEditing);
const graph = graphBuilder.compile({ checkpointer });

// 运行图并命中中断后，图将暂停。
// 使用编辑后的文本恢复它。
const threadConfig = { configurable: { thread_id: "some_id" } };
await graph.invoke(
  new Command({ resume: { editedText: "The edited text" } }),
  threadConfig
);
```

:::

??? example "Extended example: edit state with interrupt"

    :::python
    ```python
    from typing import TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define the graph state
    class State(TypedDict):
        summary: str

    # Simulate an LLM summary generation
    def generate_summary(state: State) -> State:
        return {
            "summary": "The cat sat on the mat and looked at the stars."
        }

    # Human editing node
    def human_review_edit(state: State) -> State:
        result = interrupt({
            "task": "Please review and edit the generated summary if necessary.",
            "generated_summary": state["summary"]
        })
        return {
            "summary": result["edited_summary"]
        }

    # Simulate downstream use of the edited summary
    def downstream_use(state: State) -> State:
        print(f"✅ Using edited summary: {state['summary']}")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("generate_summary", generate_summary)
    builder.add_node("human_review_edit", human_review_edit)
    builder.add_node("downstream_use", downstream_use)

    builder.set_entry_point("generate_summary")
    builder.add_edge("generate_summary", "human_review_edit")
    builder.add_edge("human_review_edit", "downstream_use")
    builder.add_edge("downstream_use", END)

    # Set up in-memory checkpointing for interrupt support
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Invoke the graph until it hits the interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)

    # Output interrupt payload
    print(result["__interrupt__"])
    # Example output:
    # > [
    # >     Interrupt(
    # >         value={
    # >             'task': 'Please review and edit the generated summary if necessary.',
    # >             'generated_summary': 'The cat sat on the mat and looked at the stars.'
    # >         },
    # >         id='...'
    # >     )
    # > ]

    # Resume the graph with human-edited input
    edited_summary = "The cat lay on the rug, gazing peacefully at the night sky."
    resumed_result = graph.invoke(
        Command(resume={"edited_summary": edited_summary}),
        config=config
    )
    print(resumed_result)
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define the graph state
    const StateAnnotation = z.object({
      summary: z.string(),
    });

    // Simulate an LLM summary generation
    function generateSummary(state: z.infer<typeof StateAnnotation>) {
      return {
        summary: "The cat sat on the mat and looked at the stars."
      };
    }

    // Human editing node
    function humanReviewEdit(state: z.infer<typeof StateAnnotation>) {
      const result = interrupt({
        task: "Please review and edit the generated summary if necessary.",
        generatedSummary: state.summary
      });
      return {
        summary: result.editedSummary
      };
    }

    // Simulate downstream use of the edited summary
    function downstreamUse(state: z.infer<typeof StateAnnotation>) {
      console.log(`✅ Using edited summary: ${state.summary}`);
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("generateSummary", generateSummary)
      .addNode("humanReviewEdit", humanReviewEdit)
      .addNode("downstreamUse", downstreamUse)
      .addEdge(START, "generateSummary")
      .addEdge("generateSummary", "humanReviewEdit")
      .addEdge("humanReviewEdit", "downstreamUse")
      .addEdge("downstreamUse", END);

    // Set up in-memory checkpointing for interrupt support
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Invoke the graph until it hits the interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    const result = await graph.invoke({}, config);

    // Output interrupt payload
    console.log(result.__interrupt__);
    // Example output:
    // [{
    //   value: {
    //     task: 'Please review and edit the generated summary if necessary.',
    //     generatedSummary: 'The cat sat on the mat and looked at the stars.'
    //   },
    //   resumable: true,
    //   ...
    // }]

    // Resume the graph with human-edited input
    const editedSummary = "The cat lay on the rug, gazing peacefully at the night sky.";
    const resumedResult = await graph.invoke(
      new Command({ resume: { editedSummary } }),
      config
    );
    console.log(resumedResult);
    ```
    :::

### 审核工具调用

<figure markdown="1">
![image](../../concepts/img/human_in_the_loop/tool-call-review.png){: style="max-height:400px"}
<figcaption>人类可以审核和编辑 LLM 的输出，然后再继续。这在 LLM 请求的工具调用可能敏感或需要人工监督的应用程序中尤其重要。
</figcaption>
</figure>

要将人工干预步骤添加到工具：

1. 在工具中使用 `interrupt()` 来暂停执行。
2. 使用 `Command` 恢复，以便根据人工输入继续。

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.types import interrupt
from langgraph.prebuilt import create_react_agent

# 一个需要人工审核/批准的敏感工具示例
def book_hotel(hotel_name: str):
    """Book a hotel"""
    # highlight-next-line
    response = interrupt( # (1)!
        f"Trying to call `book_hotel` with args {{'hotel_name': {hotel_name}}}. "
        "Please approve or suggest edits."
    )
    if response["type"] == "accept":
        pass
    elif response["type"] == "edit":
        hotel_name = response["args"]["hotel_name"]
    else:
        raise ValueError(f"Unknown response type: {response['type']}")
    return f"Successfully booked a stay at {hotel_name}."

# highlight-next-line
checkpointer = InMemorySaver() # (2)!

agent = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    tools=[book_hotel],
    # highlight-next-line
    checkpointer=checkpointer, # (3)!
)
```

1. @[`interrupt` 函数][interrupt] 在代理图中的特定节点处暂停。在此示例中，我们在工具函数开始时调用 `interrupt()`，这会在执行该工具的节点处暂停图。`interrupt()` 中的信息（例如，工具调用）可以显示给人类，并且图可以使用用户输入（工具调用批准、编辑或反馈）恢复。
2. `InMemorySaver` 用于在工具调用循环的每一步存储代理状态。这支持[短期记忆](../memory/add-memory.md#add-short-term-memory)和[人工干预](../../concepts/human_in_the_loop.md)功能。在此示例中，我们使用 `InMemorySaver` 在内存中存储代理状态。在生产应用程序中，代理状态将存储在数据库中。
3. 使用 `checkpointer` 初始化代理。
   :::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";
import { interrupt } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// 一个需要人工审核/批准的敏感工具示例
const bookHotel = tool(
  async ({ hotelName }) => {
    // highlight-next-line
    const response = interrupt( // (1)!
      // `bookHotel` 的参数包含 `hotelName`
      `Trying to call \`bookHotel\` with args {"hotelName": "${hotelName}"}. ` +
        "Please approve or suggest edits."
    );
    if (response.type === "accept") {
      // 继续使用原始参数
    } else if (response.type === "edit") {
      hotelName = response.args.hotelName;
    } else {
      throw new Error(`Unknown response type: ${response.type}`);
    }
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "bookHotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string(),
    }),
  }
);

// highlight-next-line
const checkpointer = new MemorySaver(); // (2)!

const agent = createReactAgent({
  llm: model,
  tools: [bookHotel],
  // highlight-next-line
  checkpointSaver: checkpointer, // (3)!
});
```

1. @[`interrupt` 函数][interrupt] 在代理图中的特定节点处暂停。在此示例中，我们在工具函数开始时调用 `interrupt()`，这会在执行该工具的节点处暂停图。`interrupt()` 中的信息（例如，工具调用）可以显示给人类，并且图可以使用用户输入（工具调用批准、编辑或反馈）恢复。
2. `MemorySaver` 用于在工具调用循环的每一步存储代理状态。这支持[短期记忆](../memory/add-memory.md#add-short-term-memory)和[人工干预](../../concepts/human_in_the_loop.md)功能。在此示例中，我们使用 `MemorySaver` 在内存中存储代理状态。在生产应用程序中，代理状态将存储在数据库中。
3. 使用 `checkpointSaver` 初始化代理。
   :::

使用 `stream()` 方法运行代理，并传递 `config` 对象来指定线程 ID。这允许代理在未来的调用中恢复相同的对话。

:::python

```python
config = {
   "configurable": {
      # highlight-next-line
      "thread_id": "1"
   }
}

for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "book a stay at McKittrick hotel"}]},
    # highlight-next-line
    config
):
    print(chunk)
    print("\n")
```

:::

:::js

```typescript
const config = {
  configurable: {
    // highlight-next-line
    thread_id: "1",
  },
};

const stream = await agent.stream(
  { messages: [{ role: "user", content: "book a stay at McKittrick hotel" }] },
  // highlight-next-line
  config
);

for await (const chunk of stream) {
  console.log(chunk);
  console.log("\n");
}
```

:::

> 您应该看到代理运行直到达到 `interrupt()` 调用，此时它会暂停并等待人工输入。

使用 `Command` 恢复代理，以便根据人工输入继续。

:::python

```python
from langgraph.types import Command

for chunk in agent.stream(
    # highlight-next-line
    Command(resume={"type": "accept"}),  # (1)!
    # Command(resume={"type": "edit", "args": {"hotel_name": "McKittrick Hotel"}}),
    config
):
    print(chunk)
    print("\n")
```

1. @[`interrupt` 函数][interrupt] 与 @[`Command`][Command] 对象结合使用，以恢复图并提供人类提供的值。
   :::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const resumeStream = await agent.stream(
  // highlight-next-line
  new Command({ resume: { type: "accept" } }), // (1)!
  // new Command({ resume: { type: "edit", args: { hotelName: "McKittrick Hotel" } } }),
  config
);

for await (const chunk of resumeStream) {
  console.log(chunk);
  console.log("\n");
}
```

1. @[`interrupt` 函数][interrupt] 与 @[`Command`][Command] 对象结合使用，以恢复图并提供人类提供的值。
   :::

### 为任何工具添加中断

您可以创建一个包装器来为 _所有_ 工具添加中断。下面的示例提供了一个与 [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) 和 [Agent Chat UI](https://github.com/langchain-ai/agent-chat-ui) 兼容的参考实现。

:::python

```python title="Wrapper that adds human-in-the-loop to any tool"
from typing import Callable
from langchain_core.tools import BaseTool, tool as create_tool
from langchain_core.runnables import RunnableConfig
from langgraph.types import interrupt
from langgraph.prebuilt.interrupt import HumanInterruptConfig, HumanInterrupt

def add_human_in_the_loop(
    tool: Callable | BaseTool,
    *,
    interrupt_config: HumanInterruptConfig = None,
) -> BaseTool:
    """Wrap a tool to support human-in-the-loop review."""
    if not isinstance(tool, BaseTool):
        tool = create_tool(tool)

    if interrupt_config is None:
        interrupt_config = {
            "allow_accept": True,
            "allow_edit": True,
            "allow_respond": True,
        }

    @create_tool(  # (1)!
        tool.name,
        description=tool.description,
        args_schema=tool.args_schema
    )
    def call_tool_with_interrupt(config: RunnableConfig, **tool_input):
        request: HumanInterrupt = {
            "action_request": {
                "action": tool.name,
                "args": tool_input
            },
            "config": interrupt_config,
            "description": "Please review the tool call"
        }
        # highlight-next-line
        response = interrupt([request])[0]  # (2)!
        # approve the tool call
        if response["type"] == "accept":
            tool_response = tool.invoke(tool_input, config)
        # update tool call args
        elif response["type"] == "edit":
            tool_input = response["args"]["args"]
            tool_response = tool.invoke(tool_input, config)
        # respond to the LLM with user feedback
        elif response["type"] == "response":
            user_feedback = response["args"]
            tool_response = user_feedback
        else:
            raise ValueError(f"Unsupported interrupt response type: {response['type']}")

        return tool_response

    return call_tool_with_interrupt
```

1. 此包装器创建了一个新工具，该工具在执行包装工具 _之前_ 调用 `interrupt()`。
2. `interrupt()` 使用 [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) 期望的特殊输入和输出格式： - 一系列 @[`HumanInterrupt`][HumanInterrupt] 对象被发送到 `AgentInbox` 以向最终用户渲染中断信息 - 恢复值由 `AgentInbox` 提供为列表（即 `Command(resume=[...])`）
   :::

:::js

```typescript title="Wrapper that adds human-in-the-loop to any tool"
import { StructuredTool, tool } from "@langchain/core/tools";
import { RunnableConfig } from "@langchain/core/runnables";
import { interrupt } from "@langchain/langgraph";

interface HumanInterruptConfig {
  allowAccept?: boolean;
  allowEdit?: boolean;
  allowRespond?: boolean;
}

interface HumanInterrupt {
  actionRequest: {
    action: string;
    args: Record<string, any>;
  };
  config: HumanInterruptConfig;
  description: string;
}

function addHumanInTheLoop(
  originalTool: StructuredTool,
  interruptConfig: HumanInterruptConfig = {
    allowAccept: true,
    allowEdit: true,
    allowRespond: true,
  }
): StructuredTool {
  // Wrap the original tool to support human-in-the-loop review
  return tool(
    // (1)!
    async (toolInput: Record<string, any>, config?: RunnableConfig) => {
      const request: HumanInterrupt = {
        actionRequest: {
          action: originalTool.name,
          args: toolInput,
        },
        config: interruptConfig,
        description: "Please review the tool call",
      };

      // highlight-next-line
      const response = interrupt([request])[0]; // (2)!

      // approve the tool call
      if (response.type === "accept") {
        return await originalTool.invoke(toolInput, config);
      }
      // update tool call args
      else if (response.type === "edit") {
        const updatedArgs = response.args.args;
        return await originalTool.invoke(updatedArgs, config);
      }
      // respond to the LLM with user feedback
      else if (response.type === "response") {
        return response.args;
      } else {
        throw new Error(
          `Unsupported interrupt response type: ${response.type}`
        );
      }
    },
    {
      name: originalTool.name,
      description: originalTool.description,
      schema: originalTool.schema,
    }
  );
}
```

1. 此包装器创建了一个新工具，该工具在执行包装工具 _之前_ 调用 `interrupt()`。
2. `interrupt()` 使用 [Agent Inbox UI](https://github.com/langchain-ai/agent-inbox) 期望的特殊输入和输出格式： - 一系列 [`HumanInterrupt`] 对象被发送到 `AgentInbox` 以向最终用户渲染中断信息 - 恢复值由 `AgentInbox` 提供为列表（即 `Command({ resume: [...] })`）
   :::

您可以使用此包装器将 `interrupt()` 添加到任何工具中，而无需将其 _插入_ 工具内部：

:::python

```python
from langgraph.checkpoint.memory import InMemorySaver
from langgraph.prebuilt import create_react_agent

# highlight-next-line
checkpointer = InMemorySaver()

def book_hotel(hotel_name: str):
   """Book a hotel"""
   return f"Successfully booked a stay at {hotel_name}."


agent = create_react_agent(
    model="anthropic:claude-3-5-sonnet-latest",
    tools=[
        # highlight-next-line
        add_human_in_the_loop(book_hotel), # (1)!
    ],
    # highlight-next-line
    checkpointer=checkpointer,
)

config = {"configurable": {"thread_id": "1"}}

# Run the agent
for chunk in agent.stream(
    {"messages": [{"role": "user", "content": "book a stay at McKittrick hotel"}]},
    # highlight-next-line
    config
):
    print(chunk)
    print("\n")
```

1. `add_human_in_the_loop` 包装器用于向工具添加 `interrupt()`。此功能允许代理在继续工具调用之前暂停执行并等待人工输入。
   :::

:::js

```typescript
import { MemorySaver } from "@langchain/langgraph";
import { createReactAgent } from "@langchain/langgraph/prebuilt";
import { tool } from "@langchain/core/tools";
import { z } from "zod";

// highlight-next-line
const checkpointer = new MemorySaver();

const bookHotel = tool(
  async ({ hotelName }) => {
    return `Successfully booked a stay at ${hotelName}.`;
  },
  {
    name: "bookHotel",
    description: "Book a hotel",
    schema: z.object({
      hotelName: z.string(),
    }),
  }
);

const agent = createReactAgent({
  llm: model,
  tools: [
    // highlight-next-line
    addHumanInTheLoop(bookHotel), // (1)!
  ],
  // highlight-next-line
  checkpointSaver: checkpointer,
});

const config = { configurable: { thread_id: "1" } };

// Run the agent
const stream = await agent.stream(
  { messages: [{ role: "user", content: "book a stay at McKittrick hotel" }] },
  // highlight-next-line
  config
);

for await (const chunk of stream) {
  console.log(chunk);
  console.log("\n");
}
```

1. `addHumanInTheLoop` 包装器用于向工具添加 `interrupt()`。此功能允许代理在继续工具调用之前暂停执行并等待人工输入。
   :::

> 您应该看到代理运行直到达到 `interrupt()` 调用，此时它会暂停并等待人工输入。

使用 `Command` 恢复代理，以便根据人工输入继续。

:::python

```python
from langgraph.types import Command

for chunk in agent.stream(
    # highlight-next-line
    Command(resume=[{"type": "accept"}]),
    # Command(resume=[{"type": "edit", "args": {"args": {"hotel_name": "McKittrick Hotel"}}}]),
    config
):
    print(chunk)
    print("\n")
```

:::

:::js

```typescript
import { Command } from "@langchain/langgraph";

const resumeStream = await agent.stream(
  // highlight-next-line
  new Command({ resume: [{ type: "accept" }] }),
  // new Command({ resume: [{ type: "edit", args: { args: { hotelName: "McKittrick Hotel" } } }] }),
  config
);

for await (const chunk of resumeStream) {
  console.log(chunk);
  console.log("\n");
}
```

:::

### 验证人工输入

如果您需要在图本身内（而不是在客户端）验证人类提供的输入，可以通过在单个节点中使用多个中断调用来实现。

:::python

```python
from langgraph.types import interrupt

def human_node(state: State):
    """Human node with validation."""
    question = "What is your age?"

    while True:
        answer = interrupt(question)

        # Validate answer, if the answer isn't valid ask for input again.
        if not isinstance(answer, int) or answer < 0:
            question = f"'{answer} is not a valid age. What is your age?"
            answer = None
            continue
        else:
            # If the answer is valid, we can proceed.
            break

    print(f"The human in the loop is {answer} years old.")
    return {
        "age": answer
    }
```

:::

:::js

```typescript
import { interrupt } from "@langchain/langgraph";

graphBuilder.addNode("humanNode", (state) => {
  // Human node with validation.
  let question = "What is your age?";

  while (true) {
    const answer = interrupt(question);

    // Validate answer, if the answer isn't valid ask for input again.
    if (typeof answer !== "number" || answer < 0) {
      question = `'${answer}' is not a valid age. What is your age?`;
      continue;
    } else {
      // If the answer is valid, we can proceed.
      break;
    }
  }

  console.log(`The human in the loop is ${answer} years old.`);
  return {
    age: answer,
  };
});
```

:::

??? example "Extended example: validating user input"

    :::python
    ```python
    from typing import TypedDict
    import uuid

    from langgraph.constants import START, END
    from langgraph.graph import StateGraph
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver

    # Define graph state
    class State(TypedDict):
        age: int

    # Node that asks for human input and validates it
    def get_valid_age(state: State) -> State:
        prompt = "Please enter your age (must be a non-negative integer)."

        while True:
            user_input = interrupt(prompt)

            # Validate the input
            try:
                age = int(user_input)
                if age < 0:
                    raise ValueError("Age must be non-negative.")
                break  # Valid input received
            except (ValueError, TypeError):
                prompt = f"'{user_input}' is not valid. Please enter a non-negative integer for age."

        return {"age": age}

    # Node that uses the valid input
    def report_age(state: State) -> State:
        print(f"✅ Human is {state['age']} years old.")
        return state

    # Build the graph
    builder = StateGraph(State)
    builder.add_node("get_valid_age", get_valid_age)
    builder.add_node("report_age", report_age)

    builder.set_entry_point("get_valid_age")
    builder.add_edge("get_valid_age", "report_age")
    builder.add_edge("report_age", END)

    # Create the graph with a memory checkpointer
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    # Run the graph until the first interrupt
    config = {"configurable": {"thread_id": uuid.uuid4()}}
    result = graph.invoke({}, config=config)
    print(result["__interrupt__"])  # First prompt: "Please enter your age..."

    # Simulate an invalid input (e.g., string instead of integer)
    result = graph.invoke(Command(resume="not a number"), config=config)
    print(result["__interrupt__"])  # Follow-up prompt with validation message

    # Simulate a second invalid input (e.g., negative number)
    result = graph.invoke(Command(resume="-10"), config=config)
    print(result["__interrupt__"])  # Another retry

    # Provide valid input
    final_result = graph.invoke(Command(resume="25"), config=config)
    print(final_result)  # Should include the valid age
    ```
    :::

    :::js
    ```typescript
    import { z } from "zod";
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      END,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";

    // Define graph state
    const StateAnnotation = z.object({
      age: z.number(),
    });

    // Node that asks for human input and validates it
    function getValidAge(state: z.infer<typeof StateAnnotation>) {
      let prompt = "Please enter your age (must be a non-negative integer).";

      while (true) {
        const userInput = interrupt(prompt);

        // Validate the input
        try {
          const age = parseInt(userInput as string);
          if (isNaN(age) || age < 0) {
            throw new Error("Age must be non-negative.");
          }
          return { age };
        } catch (error) {
          prompt = `'${userInput}' is not valid. Please enter a non-negative integer for age.`;
        }
      }
    }

    // Node that uses the valid input
    function reportAge(state: z.infer<typeof StateAnnotation>) {
      console.log(`✅ Human is ${state.age} years old.`);
      return state;
    }

    // Build the graph
    const builder = new StateGraph(StateAnnotation)
      .addNode("getValidAge", getValidAge)
      .addNode("reportAge", reportAge)
      .addEdge(START, "getValidAge")
      .addEdge("getValidAge", "reportAge")
      .addEdge("reportAge", END);

    // Create the graph with a memory checkpointer
    const checkpointer = new MemorySaver();
    const graph = builder.compile({ checkpointer });

    // Run the graph until the first interrupt
    const config = { configurable: { thread_id: uuidv4() } };
    let result = await graph.invoke({}, config);
    console.log(result.__interrupt__);  // First prompt: "Please enter your age..."

    // Simulate an invalid input (e.g., string instead of integer)
    result = await graph.invoke(new Command({ resume: "not a number" }), config);
    console.log(result.__interrupt__);  // Follow-up prompt with validation message

    // Simulate a second invalid input (e.g., negative number)
    result = await graph.invoke(new Command({ resume: "-10" }), config);
    console.log(result.__interrupt__);  // Another retry

    // Provide valid input
    const finalResult = await graph.invoke(new Command({ resume: "25" }), config);
    console.log(finalResult);  // Should include the valid age
    ```
    :::

:::python

## 使用中断进行调试

要调试和测试图，请使用[静态中断](../../concepts/human_in_the_loop.md#key-capabilities)（也称为静态断点）来逐个节点地步进图执行或在特定节点处暂停图执行。静态中断在节点执行之前或之后定义的点触发。您可以通过在编译时或运行时指定 `interrupt_before` 和 `interrupt_after` 来设置静态中断。

!!! warning

    静态中断 **不** 推荐用于人工干预工作流。请使用[动态中断](#pause-using-interrupt)。

=== "Compile time"

    ```python
    # highlight-next-line
    graph = graph_builder.compile( # (1)!
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"], # (3)!
        checkpointer=checkpointer, # (4)!
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=thread_config) # (5)!

    # Resume the graph
    graph.invoke(None, config=thread_config) # (6)!
    ```

    1. 断点在 `compile` 时间设置。
    2. `interrupt_before` 指定在节点执行前暂停执行的节点。
    3. `interrupt_after` 指定在节点执行后暂停执行的节点。
    4. 使用检查点（checkpointer）来启用断点。
    5. 图运行直到命中第一个断点。
    6. 通过传递 `None` 作为输入来恢复图。这将运行图直到命中下一个断点。

=== "Run time"

    ```python
    # highlight-next-line
    graph.invoke( # (1)!
        inputs,
        # highlight-next-line
        interrupt_before=["node_a"], # (2)!
        # highlight-next-line
        interrupt_after=["node_b", "node_c"] # (3)!
        config={
            "configurable": {"thread_id": "some_thread"}
        },
    )

    config = {
        "configurable": {
            "thread_id": "some_thread"
        }
    }

    # Run the graph until the breakpoint
    graph.invoke(inputs, config=config) # (4)!

    # Resume the graph
    graph.invoke(None, config=config) # (5)!
    ```

    1. `graph.invoke` 使用 `interrupt_before` 和 `interrupt_after` 参数调用。这是运行时配置，可以针对每次调用进行更改。
    2. `interrupt_before` 指定在节点执行前暂停执行的节点。
    3. `interrupt_after` 指定在节点执行后暂停执行的节点。
    4. 图运行直到命中第一个断点。
    5. 通过传递 `None` 作为输入来恢复图。这将运行图直到命中下一个断点。

    !!! note

        您不能在运行时为 **子图** 设置静态断点。
        如果您有子图，则必须在编译时设置断点。

??? example "Setting static breakpoints"

    ```python
    from IPython.display import Image, display
    from typing_extensions import TypedDict

    from langgraph.checkpoint.memory import InMemorySaver
    from langgraph.graph import StateGraph, START, END


    class State(TypedDict):
        input: str


    def step_1(state):
        print("---Step 1---")
        pass


    def step_2(state):
        print("---Step 2---")
        pass


    def step_3(state):
        print("---Step 3---")
        pass


    builder = StateGraph(State)
    builder.add_node("step_1", step_1)
    builder.add_node("step_2", step_2)
    builder.add_node("step_3", step_3)
    builder.add_edge(START, "step_1")
    builder.add_edge("step_1", "step_2")
    builder.add_edge("step_2", "step_3")
    builder.add_edge("step_3", END)

    # Set up a checkpointer
    checkpointer = InMemorySaver() # (1)!

    graph = builder.compile(
        checkpointer=checkpointer, # (2)!
        interrupt_before=["step_3"] # (3)!
    )

    # View
    display(Image(graph.get_graph().draw_mermaid_png()))


    # Input
    initial_input = {"input": "hello world"}

    # Thread
    thread = {"configurable": {"thread_id": "1"}}

    # Run the graph until the first interruption
    for event in graph.stream(initial_input, thread, stream_mode="values"):
        print(event)

    # This will run until the breakpoint
    # You can get the state of the graph at this point
    print(graph.get_state(config))

    # You can continue the graph execution by passing in `None` for the input
    for event in graph.stream(None, thread, stream_mode="values"):
        print(event)
    ```

### 在 LangGraph Studio 中使用静态中断

您可以使用[LangGraph Studio](../../concepts/langgraph_studio.md) 来调试您的图。您可以在 UI 中设置静态断点，然后运行图。您还可以使用 UI 在执行的任何点检查图状态。

![image](../../concepts/img/human_in_the_loop/static-interrupt.png){: style="max-height:400px"}

使用 `langgraph dev` 部署[本地应用程序](../../tutorials/langgraph-platform/local-server.md)时，LangGraph Studio 是免费的。

:::

## 考虑因素

在使用人工干预时，需要注意一些事项。

### 使用有副作用的代码

将具有副作用的代码（例如 API 调用）放在 `interrupt` 之后或单独的节点中，以避免重复执行，因为每次恢复节点时都会重新触发它们。

=== "Side effects after interrupt"

    :::python
    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""

        answer = interrupt(question)

        api_call(answer) # OK as it's after the interrupt
    ```
    :::

    :::js
    ```typescript
    import { interrupt } from "@langchain/langgraph";

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      // Human node with validation.

      const answer = interrupt(question);

      apiCall(answer); // OK as it's after the interrupt
    }
    ```
    :::

=== "Side effects in a separate node"

    :::python
    ```python
    from langgraph.types import interrupt

    def human_node(state: State):
        """Human node with validation."""

        answer = interrupt(question)

        return {
            "answer": answer
        }

    def api_call_node(state: State):
        api_call(...) # OK as it's in a separate node
    ```
    :::

    :::js
    ```typescript
    import { interrupt } from "@langchain/langgraph";

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      // Human node with validation.

      const answer = interrupt(question);

      return {
        answer
      };
    }

    function apiCallNode(state: z.infer<typeof StateAnnotation>) {
      apiCall(state.answer); # OK as it's in a separate node
    }
    ```
    :::

### 与作为函数调用的子图一起使用

在将子图作为函数调用时，父图将从调用子图的节点的 _开头_ 恢复执行（当 `interrupt` 被触发时）。同样，**子图** 将从调用 `interrupt()` 函数的节点的 _开头_ 恢复执行。

:::python

```python
def node_in_parent_graph(state: State):
    some_code()  # <-- This will re-execute when the subgraph is resumed.
    # Invoke a subgraph as a function.
    # The subgraph contains an `interrupt` call.
    subgraph_result = subgraph.invoke(some_input)
    ...
```

:::

:::js

```typescript
async function nodeInParentGraph(state: z.infer<typeof StateAnnotation>) {
  someCode(); # <-- This will re-execute when the subgraph is resumed.
  # Invoke a subgraph as a function.
  # The subgraph contains an `interrupt` call.
  const subgraphResult = await subgraph.invoke(someInput);
  # ...
}
```

:::

??? example "Extended example: parent and subgraph execution flow"

    假设我们有一个包含 3 个节点的父图：

    **父图**: `node_1` → `node_2` (子图调用) → `node_3`

    子图包含 3 个节点，其中第二个节点包含一个 `interrupt`：

    **子图**: `sub_node_1` → `sub_node_2` (`interrupt`) → `sub_node_3`

    恢复图时，执行将按以下方式进行：

    1. **跳过父图中的 `node_1`**（已执行，图状态已保存在快照中）。
    2. **重新执行父图中的 `node_2`** 从开头。
    3. **跳过子图中的 `sub_node_1`**（已执行，图状态已保存在快照中）。
    4. **从开头重新执行子图中的 `sub_node_2`**。
    5. 继续执行 `sub_node_3` 及后续节点。

    以下是您可以用来理解子图如何与中断配合使用的简短示例代码。
    它会计算入口次数并打印计数。

    :::python
    ```python
    import uuid
    from typing import TypedDict

    from langgraph.graph import StateGraph
    from langgraph.constants import START
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver


    class State(TypedDict):
        """The graph state."""
        state_counter: int


    counter_node_in_subgraph = 0

    def node_in_subgraph(state: State):
        """A node in the sub-graph."""
        global counter_node_in_subgraph
        counter_node_in_subgraph += 1  # This code will **NOT** run again!
        print(f"Entered `node_in_subgraph` a total of {counter_node_in_subgraph} times")

    counter_human_node = 0

    def human_node(state: State):
        global counter_human_node
        counter_human_node += 1 # This code will run again!
        print(f"Entered human_node in sub-graph a total of {counter_human_node} times")
        answer = interrupt("what is your name?")
        print(f"Got an answer of {answer}")


    checkpointer = InMemorySaver()

    subgraph_builder = StateGraph(State)
    subgraph_builder.add_node("some_node", node_in_subgraph)
    subgraph_builder.add_node("human_node", human_node)
    subgraph_builder.add_edge(START, "some_node")
    subgraph_builder.add_edge("some_node", "human_node")
    subgraph = subgraph_builder.compile(checkpointer=checkpointer)


    counter_parent_node = 0

    def parent_node(state: State):
        """This parent node will invoke the subgraph."""
        global counter_parent_node

        counter_parent_node += 1 # This code will run again on resuming!
        print(f"Entered `parent_node` a total of {counter_parent_node} times")

        # Please note that we're intentionally incrementing the state counter
        # in the graph state as well to demonstrate that the subgraph update
        # of the same key will not conflict with the parent graph (until
        subgraph_state = subgraph.invoke(state)
        return subgraph_state


    builder = StateGraph(State)
    builder.add_node("parent_node", parent_node)
    builder.add_edge(START, "parent_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
          "thread_id": uuidv4(),
        }
    }

    for chunk in graph.stream({"state_counter": 1}, config):
        print(chunk)

    print('--- Resuming ---')

    for chunk in graph.stream(Command(resume="35"), config):
        print(chunk)
    ```

    This will print out

    ```pycon
    Entered `parent_node` a total of 1 times
    Entered `node_in_subgraph` a total of 1 times
    Entered human_node in sub-graph a total of 1 times
    {'__interrupt__': (Interrupt(value='what is your name?', id='...'),)}
    --- Resuming ---
    Entered `parent_node` a total of 2 times
    Entered human_node in sub-graph a total of 2 times
    Got an answer of 35
    {'parent_node': {'state_counter': 1}}
    ```
    :::

    :::js
    ```typescript
    import { v4 as uuidv4 } from "uuid";
    import {
      StateGraph,
      START,
      interrupt,
      Command,
      MemorySaver
    } from "@langchain/langgraph";
    import { z } from "zod";

    const StateAnnotation = z.object({
      stateCounter: z.number(),
    });

    // Global variable to track the number of attempts
    let counterNodeInSubgraph = 0;

    function nodeInSubgraph(state: z.infer<typeof StateAnnotation>) {
      // A node in the sub-graph.
      counterNodeInSubgraph += 1; # This code will **NOT** run again!
      console.log(`Entered 'nodeInSubgraph' a total of ${counterNodeInSubgraph} times`);
      return {};
    }

    let counterHumanNode = 0;

    function humanNode(state: z.infer<typeof StateAnnotation>) {
      counterHumanNode += 1; # This code will run again!
      console.log(`Entered humanNode in sub-graph a total of ${counterHumanNode} times`);
      const answer = interrupt("what is your name?");
      console.log(`Got an answer of ${answer}`);
      return {};
    }

    const checkpointer = new MemorySaver();

    const subgraphBuilder = new StateGraph(StateAnnotation)
      .addNode("someNode", nodeInSubgraph)
      .addNode("humanNode", humanNode)
      .addEdge(START, "someNode")
      .addEdge("someNode", "humanNode");
    const subgraph = subgraphBuilder.compile({ checkpointer });

    let counterParentNode = 0;

    async function parentNode(state: z.infer<typeof StateAnnotation>) {
      # This parent node will invoke the subgraph.
      counterParentNode += 1; # This code will run again on resuming!
      console.log(`Entered 'parentNode' a total of ${counterParentNode} times`);

      # Please note that we're intentionally incrementing the state counter
      # in the graph state as well to demonstrate that the subgraph update
      # of the same key will not conflict with the parent graph (until
      const subgraphState = await subgraph.invoke(state);
      return subgraphState;
    }

    const builder = new StateGraph(StateAnnotation)
      .addNode("parentNode", parentNode)
      .addEdge(START, "parentNode");

    # A checkpointer must be enabled for interrupts to work!
    const graph = builder.compile({ checkpointer });

    const config = {
      configurable: {
        thread_id: uuidv4(),
      }
    };

    const stream = await graph.stream({ stateCounter: 1 }, config);
    for await (const chunk of stream) {
      console.log(chunk);
    }

    console.log('--- Resuming ---');

    const resumeStream = await graph.stream(new Command({ resume: "35" }), config);
    for await (const chunk of resumeStream) {
      console.log(chunk);
    }
    ```

    This will print out

    ```
    Entered 'parentNode' a total of 1 times
    Entered 'nodeInSubgraph' a total of 1 times
    Entered humanNode in sub-graph a total of 1 times
    { __interrupt__: [{ value: 'what is your name?', resumable: true, ns: ['parentNode:4c3a0248-21f0-1287-eacf-3002bc304db4', 'humanNode:2fe86d52-6f70-2a3f-6b2f-b1eededd6348'], when: 'during' }] }
    --- Resuming ---
    Entered 'parentNode' a total of 2 times
    Entered humanNode in sub-graph a total of 2 times
    Got an answer of 35
    { parentNode: null }
    ```
    :::

### 在单个节点中使用多个中断

在 _单个_ 节点中使用多个中断可能有助于实现类似[验证人工输入](#validate-human-input)的模式。但是，如果处理不当，在单个节点中使用多个中断可能会导致意外行为。

当节点包含多个中断调用时，LangGraph 会为执行该节点的任务维护一个针对恢复值的列表。每次恢复执行时，都会从节点的开头开始。对于遇到的每个中断，LangGraph 会检查任务的恢复列表中是否存在匹配的值。匹配是 **严格基于索引的**，因此节点内的中断调用顺序至关重要。

为避免问题，请避免在执行之间动态更改节点结构。这包括添加、删除或重新排序中断调用，因为此类更改可能导致索引不匹配。这些问题通常源于非标准的模式，例如通过 `Command(resume=..., update=SOME_STATE_MUTATION)` 改变状态，或依赖全局变量动态修改节点结构。

:::python
??? example "Extended example: incorrect code that introduces non-determinism"

    ```python
    import uuid
    from typing import TypedDict, Optional

    from langgraph.graph import StateGraph
    from langgraph.constants import START
    from langgraph.types import interrupt, Command
    from langgraph.checkpoint.memory import InMemorySaver


    class State(TypedDict):
        """The graph state."""

        age: Optional[str]
        name: Optional[str]


    def human_node(state: State):
        if not state.get('name'):
            name = interrupt("what is your name?")
        else:
            name = "N/A"

        if not state.get('age'):
            age = interrupt("what is your age?")
        else:
            age = "N/A"

        print(f"Name: {name}. Age: {age}")

        return {
            "age": age,
            "name": name,
        }


    builder = StateGraph(State)
    builder.add_node("human_node", human_node)
    builder.add_edge(START, "human_node")

    # A checkpointer must be enabled for interrupts to work!
    checkpointer = InMemorySaver()
    graph = builder.compile(checkpointer=checkpointer)

    config = {
        "configurable": {
            "thread_id": uuid.uuid4(),
        }
    }

    for chunk in graph.stream({"age": None, "name": None}, config):
        print(chunk)

    for chunk in graph.stream(Command(resume="John", update={"name": "foo"}), config):
        print(chunk)
    ```

    ```pycon
    {'__interrupt__': (Interrupt(value='what is your name?', id='...'),)}
    Name: N/A. Age: John
    {'human_node': {'age': 'John', 'name': 'N/A'}}
    ```

:::