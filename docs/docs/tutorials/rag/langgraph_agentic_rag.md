# 智能 RAG

在本教程中，我们将构建一个 [检索式代理](https://python.langchain.com/docs/tutorials/qa_chat_history)。当您希望 LLM 就何时从向量存储中检索上下文或直接响应用户做出决定时，检索式代理会非常有用。

在本教程结束时，我们将完成以下工作：

1. 获取并预处理用于检索的文档。
2. 为代理中的语义搜索对这些文档进行索引并创建检索器工具。
3. 构建一个智能 RAG 系统，该系统可以决定何时使用检索器工具。

![Screenshot 2024-02-14 at 3.43.58 PM.png](assets/screenshot_2024_02_14_3_43_58_pm.png)

## 设置

让我们下载所需的软件包并设置我们的 API 密钥：

```python
%%capture --no-stderr
%pip install -U --quiet langgraph "langchain[openai]" langchain-community langchain-text-splitters
```

```python
import getpass
import os


def _set_env(key: str):
    if key not in os.environ:
        os.environ[key] = getpass.getpass(f"{key}:")


_set_env("OPENAI_API_KEY")
```

!!! tip
    注册 LangSmith 以快速发现问题并提高您的 LangGraph 项目的性能。[LangSmith](https://docs.smith.langchain.com) 可让您使用跟踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用。


## 1. 预处理文档

1. 获取用于我们 RAG 系统的文档。我们将使用 Lilian Weng 的精美博客 [的最近三页](https://lilianweng.github.io/)。我们将首先使用 `WebBaseLoader` 实用程序获取页面的内容：

    ```python
    from langchain_community.document_loaders import WebBaseLoader

    urls = [
        "https://lilianweng.github.io/posts/2024-11-28-reward-hacking/",
        "https://lilianweng.github.io/posts/2024-07-07-hallucination/",
        "https://lilianweng.github.io/posts/2024-04-12-diffusion-video/",
    ]

    docs = [WebBaseLoader(url).load() for url in urls]
    ```

    ```python
    docs[0][0].page_content.strip()[:1000]
    ```

2. 将获取的文档分割成更小的块，以便索引到我们的向量存储中：

    ```python
    from langchain_text_splitters import RecursiveCharacterTextSplitter

    docs_list = [item for sublist in docs for item in sublist]

    text_splitter = RecursiveCharacterTextSplitter.from_tiktoken_encoder(
        chunk_size=100, chunk_overlap=50
    )
    doc_splits = text_splitter.split_documents(docs_list)
    ```

    ```python
    doc_splits[0].page_content.strip()
    ```

## 2. 创建检索器工具

现在我们有了分割好的文档，我们可以将它们索引到我们将用于语义搜索的向量存储中。

1. 使用内存向量存储和 OpenAI embeddings：

    ```python
    from langchain_core.vectorstores import InMemoryVectorStore
    from langchain_openai import OpenAIEmbeddings

    vectorstore = InMemoryVectorStore.from_documents(
        documents=doc_splits, embedding=OpenAIEmbeddings()
    )
    retriever = vectorstore.as_retriever()
    ```

2. 使用 LangChain 的预构建 `create_retriever_tool` 创建检索器工具：

    ```python
    from langchain.tools.retriever import create_retriever_tool

    retriever_tool = create_retriever_tool(
        retriever,
        "retrieve_blog_posts",
        "Search and return information about Lilian Weng blog posts.",
    )
    ```

3. 测试该工具：

    ```python
    retriever_tool.invoke({"query": "types of reward hacking"})
    ```

## 3. 生成查询

现在我们将开始为我们的智能 RAG 图构建组件（[节点](../../concepts/low_level.md#nodes) 和 [边](../../concepts/low_level.md#edges)）。请注意，组件将在 [`MessagesState`](../../concepts/low_level.md#messagesstate) 上运行——一个包含 `messages` 键（其中包含一个 [聊天消息](https://python.langchain.com/docs/concepts/messages/) 列表）的图状态。

1. 构建一个 `generate_query_or_respond` 节点。它将调用 LLM 根据当前图状态（消息列表）生成响应。根据输入的消息，它将决定使用检索器工具进行检索，或直接响应用户。请注意，我们通过 `.bind_tools` 将之前创建的 `retriever_tool` 提供给聊天模型：

    ```python
    from langgraph.graph import MessagesState
    from langchain.chat_models import init_chat_model

    response_model = init_chat_model("openai:gpt-4.1", temperature=0)


    def generate_query_or_respond(state: MessagesState):
        """根据当前状态调用模型生成响应。根据问题，它将决定使用检索器工具进行检索，或仅响应用户。
        """
        response = (
            response_model
            # highlight-next-line
            .bind_tools([retriever_tool]).invoke(state["messages"])
        )
        return {"messages": [response]}
    ```

2. 在随机输入上试用一下：

    ```python
    input = {"messages": [{"role": "user", "content": "hello!"}]}
    generate_query_or_respond(input)["messages"][-1].pretty_print()
    ```

    **输出：**
    ```
    ================================== Ai Message ==================================

    Hello! How can I help you today?
    ```

3. 问一个需要语义搜索的问题：

    ```python
    input = {
        "messages": [
            {
                "role": "user",
                "content": "What does Lilian Weng say about types of reward hacking?",
            }
        ]
    }
    generate_query_or_respond(input)["messages"][-1].pretty_print()
    ```

    **输出：**
    ```
    ================================== Ai Message ==================================
    Tool Calls:
    retrieve_blog_posts (call_tYQxgfIlnQUDMdtAhdbXNwIM)
    Call ID: call_tYQxgfIlnQUDMdtAhdbXNwIM
    Args:
        query: types of reward hacking
    ```

## 4. 评估文档

1. 添加一个 [条件边](../../concepts/low_level.md#conditional-edges)— `grade_documents` — 以确定检索到的文档是否与问题相关。我们将使用具有结构化输出模式 `GradeDocuments` 的模型来评估文档。 `grade_documents` 函数将根据评估决策（`generate_answer` 或 `rewrite_question`）返回要转到的节点名称：

    ```python
    from pydantic import BaseModel, Field
    from typing import Literal

    GRADE_PROMPT = (
        "You are a grader assessing relevance of a retrieved document to a user question. \n "
        "Here is the retrieved document: \n\n {context} \n\n"
        "Here is the user question: {question} \n"
        "If the document contains keyword(s) or semantic meaning related to the user question, grade it as relevant. \n"
        "Give a binary score 'yes' or 'no' score to indicate whether the document is relevant to the question."
    )


    # highlight-next-line
    class GradeDocuments(BaseModel):
        """Grade documents using a binary score for relevance check."""

        binary_score: str = Field(
            description="Relevance score: 'yes' if relevant, or 'no' if not relevant"
        )


    grader_model = init_chat_model("openai:gpt-4.1", temperature=0)


    def grade_documents(
        state: MessagesState,
    ) -> Literal["generate_answer", "rewrite_question"]:
        """Determine whether the retrieved documents are relevant to the question."""
        question = state["messages"][0].content
        context = state["messages"][-1].content

        prompt = GRADE_PROMPT.format(question=question, context=context)
        response = (
            grader_model
            # highlight-next-line
            .with_structured_output(GradeDocuments).invoke(
                [{"role": "user", "content": prompt}]
            )
        )
        score = response.binary_score

        if score == "yes":
            return "generate_answer"
        else:
            return "rewrite_question"
    ```

2. 使用工具响应中的不相关文档运行此代码：

    ```python
    from langchain_core.messages import convert_to_messages

    input = {
        "messages": convert_to_messages(
            [
                {
                    "role": "user",
                    "content": "What does Lilian Weng say about types of reward hacking?",
                },
                {
                    "role": "assistant",
                    "content": "",
                    "tool_calls": [
                        {
                            "id": "1",
                            "name": "retrieve_blog_posts",
                            "args": {"query": "types of reward hacking"},
                        }
                    ],
                },
                {"role": "tool", "content": "meow", "tool_call_id": "1"},
            ]
        )
    }
    grade_documents(input)
    ```

3. 确认相关文档被正确分类：

    ```python
    input = {
        "messages": convert_to_messages(
            [
                {
                    "role": "user",
                    "content": "What does Lilian Weng say about types of reward hacking?",
                },
                {
                    "role": "assistant",
                    "content": "",
                    "tool_calls": [
                        {
                            "id": "1",
                            "name": "retrieve_blog_posts",
                            "args": {"query": "types of reward hacking"},
                        }
                    ],
                },
                {
                    "role": "tool",
                    "content": "reward hacking can be categorized into two types: environment or goal misspecification, and reward tampering",
                    "tool_call_id": "1",
                },
            ]
        )
    }
    grade_documents(input)
    ```

## 5. 重写问题

1. 构建 `rewrite_question` 节点。检索器工具可以返回可能不相关的文档，这表明需要改进原始用户问题。为此，我们将调用 `rewrite_question` 节点：

    ```python
    REWRITE_PROMPT = (
        "Look at the input and try to reason about the underlying semantic intent / meaning.\n"
        "Here is the initial question:"
        "\n ------- \n"
        "{question}"
        "\n ------- \n"
        "Formulate an improved question:"
    )


    def rewrite_question(state: MessagesState):
        """Rewrite the original user question."""
        messages = state["messages"]
        question = messages[0].content
        prompt = REWRITE_PROMPT.format(question=question)
        response = response_model.invoke([{"role": "user", "content": prompt}])
        return {"messages": [{"role": "user", "content": response.content}]}
    ```

2. 试用一下：

    ```python
    input = {
        "messages": convert_to_messages(
            [
                {
                    "role": "user",
                    "content": "What does Lilian Weng say about types of reward hacking?",
                },
                {
                    "role": "assistant",
                    "content": "",
                    "tool_calls": [
                        {
                            "id": "1",
                            "name": "retrieve_blog_posts",
                            "args": {"query": "types of reward hacking"},
                        }
                    ],
                },
                {"role": "tool", "content": "meow", "tool_call_id": "1"},
            ]
        )
    }

    response = rewrite_question(input)
    print(response["messages"][-1]["content"])
    ```

    **输出：**
    ```
    What are the different types of reward hacking described by Lilian Weng, and how does she explain them?
    ```

## 6. 生成答案

1. 构建 `generate_answer` 节点：如果我们通过了评估器检查，我们可以根据原始问题和检索到的上下文生成最终答案：

    ```python
    GENERATE_PROMPT = (
        "You are an assistant for question-answering tasks. "
        "Use the following pieces of retrieved context to answer the question. "
        "If you don't know the answer, just say that you don't know. "
        "Use three sentences maximum and keep the answer concise.\n"
        "Question: {question} \n"
        "Context: {context}"
    )


    def generate_answer(state: MessagesState):
        """Generate an answer."""
        question = state["messages"][0].content
        context = state["messages"][-1].content
        prompt = GENERATE_PROMPT.format(question=question, context=context)
        response = response_model.invoke([{"role": "user", "content": prompt}])
        return {"messages": [response]}
    ```

2. 试用一下：

    ```python
    input = {
        "messages": convert_to_messages(
            [
                {
                    "role": "user",
                    "content": "What does Lilian Weng say about types of reward hacking?",
                },
                {
                    "role": "assistant",
                    "content": "",
                    "tool_calls": [
                        {
                            "id": "1",
                            "name": "retrieve_blog_posts",
                            "args": {"query": "types of reward hacking"},
                        }
                    ],
                },
                {
                    "role": "tool",
                    "content": "reward hacking can be categorized into two types: environment or goal misspecification, and reward tampering",
                    "tool_call_id": "1",
                },
            ]
        )
    }

    response = generate_answer(input)
    response["messages"][-1].pretty_print()
    ```

    **输出：**
    ```
    ================================== Ai Message ==================================

    Lilian Weng categorizes reward hacking into two types: environment or goal misspecification, and reward tampering. She considers reward hacking as a broad concept that includes both of these categories. Reward hacking occurs when an agent exploits flaws or ambiguities in the reward function to achieve high rewards without performing the intended behaviors.
    ```

## 7. 组装图

* 从 `generate_query_or_respond` 开始，并确定我们是否需要调用 `retriever_tool`
* 使用 `tools_condition` 路由到下一步：
    * 如果 `generate_query_or_respond` 返回 `tool_calls`，则调用 `retriever_tool` 来检索上下文
    * 否则，直接响应用户
* 评估检索到的文档内容与问题的相关性（`grade_documents`），并路由到下一步：
    * 如果不相关，则使用 `rewrite_question` 重写问题，然后再次调用 `generate_query_or_respond`
    * 如果相关，则继续执行 `generate_answer`，并使用包含检索到的文档上下文的 `ToolMessage` 生成最终响应

```python
from langgraph.graph import StateGraph, START, END
from langgraph.prebuilt import ToolNode
from langgraph.prebuilt import tools_condition

workflow = StateGraph(MessagesState)

# Define the nodes we will cycle between
workflow.add_node(generate_query_or_respond)
workflow.add_node("retrieve", ToolNode([retriever_tool]))
workflow.add_node(rewrite_question)
workflow.add_node(generate_answer)

workflow.add_edge(START, "generate_query_or_respond")

# Decide whether to retrieve
workflow.add_conditional_edges(
    "generate_query_or_respond",
    # Assess LLM decision (call `retriever_tool` tool or respond to the user)
    tools_condition,
    {
        # Translate the condition outputs to nodes in our graph
        "tools": "retrieve",
        END: END,
    },
)

# Edges taken after the `action` node is called.
workflow.add_conditional_edges(
    "retrieve",
    # Assess agent decision
    grade_documents,
)
workflow.add_edge("generate_answer", END)
workflow.add_edge("rewrite_question", "generate_query_or_respond")

# Compile
graph = workflow.compile()
```

可视化图：

```python
from IPython.display import Image, display

display(Image(graph.get_graph().draw_mermaid_png()))
```

![Graph](assets/agentic-rag-output.png)

## 8. 运行智能 RAG

```python
for chunk in graph.stream(
    {
        "messages": [
            {
                "role": "user",
                "content": "What does Lilian Weng say about types of reward hacking?",
            }
        ]
    }
):
    for node, update in chunk.items():
        print("Update from node", node)
        update["messages"][-1].pretty_print()
        print("\n\n")
```

**输出：**
```
Update from node generate_query_or_respond
================================== Ai Message ==================================
Tool Calls:
  retrieve_blog_posts (call_NYu2vq4km9nNNEFqJwefWKu1)
 Call ID: call_NYu2vq4km9nNNEFqJwefWKu1
  Args:
    query: types of reward hacking



Update from node retrieve
================================= Tool Message ==================================
Name: retrieve_blog_posts

(Note: Some work defines reward tampering as a distinct category of misalignment behavior from reward hacking. But I consider reward hacking as a broader concept here.)
At a high level, reward hacking can be categorized into two types: environment or goal misspecification, and reward tampering.

Why does Reward Hacking Exist?#

Pan et al. (2022) investigated reward hacking as a function of agent capabilities, including (1) model size, (2) action space resolution, (3) observation space noise, and (4) training time. They also proposed a taxonomy of three types of misspecified proxy rewards:

Let's Define Reward Hacking#
Reward shaping in RL is challenging. Reward hacking occurs when an RL agent exploits flaws or ambiguities in the reward function to obtain high rewards without genuinely learning the intended behaviors or completing the task as designed. In recent years, several related concepts have been proposed, all referring to some form of reward hacking:



Update from node generate_answer
================================== Ai Message ==================================

Lilian Weng categorizes reward hacking into two types: environment or goal misspecification, and reward tampering. She considers reward hacking as a broad concept that includes both of these categories. Reward hacking occurs when an agent exploits flaws or ambiguities in the reward function to achieve high rewards without performing the intended behaviors.
```