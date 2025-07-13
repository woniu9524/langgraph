# 构建一个 SQL 代理

在本教程中，我们将介绍如何构建一个能够回答有关 SQL 数据库问题的代理。

总体而言，该代理将执行以下操作：

1. 从数据库中获取可用表
2. 决定哪些表与问题相关
3. 获取相关表的架构
4. 基于问题和架构信息生成查询
5. 使用 LLM 双重检查查询是否存在常见错误
6. 执行查询并返回结果
7. 直到查询成功，修正数据库引擎报告的错误
8. 根据结果制定响应

!!! warning "安全提示"
    构建 SQL 数据库的问答系统需要执行模型生成的 SQL 查询。这存在固有风险。请确保您的数据库连接权限始终尽可能地限制在代理的最低需求范围内。这将减轻（但不能完全消除）构建模型驱动系统的风险。

## 1. 设置

我们先安装一些依赖项。本教程使用了来自 [langchain-community](https://python.langchain.com/docs/concepts/architecture/#langchain-community) 的 SQL 数据库和工具抽象。我们还需要一个 LangChain 的 [chat model](https://python.langchain.com/docs/concepts/chat_models/)。

```python
%%capture --no-stderr
%pip install -U langgraph langchain_community "langchain[openai]"
```

!!! tip
    注册 LangSmith，以便快速发现问题并提高 LangGraph 项目的性能。[LangSmith](https://docs.smith.langchain.com) 允许您使用跟踪数据来调试、测试和监控您使用 LangGraph 构建的 LLM 应用。

### 选择一个 LLM

首先，我们[初始化我们的 LLM](https://python.langchain.com/docs/how_to/chat_models_universal_init/)。任何支持[工具调用](https://python.langchain.com/docs/integrations/chat/#featured-providers)的模型都应该可以工作。我们在下面使用 OpenAI。

```python
from langchain.chat_models import init_chat_model

llm = init_chat_model("openai:gpt-4.1")
```

### 配置数据库

我们将为本教程创建一个 SQLite 数据库。SQLite 是一个轻量级数据库，易于设置和使用。我们将加载 `chinook` 数据库，这是一个代表数字媒体商店的示例数据库。
在此[处](https://www.sqlitetutorial.net/sqlite-sample-database/)查找有关该数据库的更多信息。

为了方便起见，我们已将数据库 (`Chinook.db`) 托管在公共 GCS 存储桶中。

```python
import requests

url = "https://storage.googleapis.com/benchmarks-artifacts/chinook/Chinook.db"

response = requests.get(url)

if response.status_code == 200:
    # Open a local file in binary write mode
    with open("Chinook.db", "wb") as file:
        # Write the content of the response (the file) to the local file
        file.write(response.content)
    print("File downloaded and saved as Chinook.db")
else:
    print(f"Failed to download the file. Status code: {response.status_code}")
```

我们将使用 `langchain_community` 包中提供的便捷 SQL 数据库包装器与数据库进行交互。该包装器提供了执行 SQL 查询和获取结果的简单接口：

```python
from langchain_community.utilities import SQLDatabase

db = SQLDatabase.from_uri("sqlite:///Chinook.db")

print(f"Dialect: {db.dialect}")
print(f"Available tables: {db.get_usable_table_names()}")
print(f'Sample output: {db.run("SELECT * FROM Artist LIMIT 5;")}')
```

**输出:**
```
Dialect: sqlite
Available tables: ['Album', 'Artist', 'Customer', 'Employee', 'Genre', 'Invoice', 'InvoiceLine', 'MediaType', 'Playlist', 'PlaylistTrack', 'Track']
Sample output: [(1, 'AC/DC'), (2, 'Accept'), (3, 'Aerosmith'), (4, 'Alanis Morissette'), (5, 'Alice In Chains')]
```

### 用于数据库交互的工具

`langchain-community` 实现了与我们的 `SQLDatabase` 交互的一些内置工具，包括列出表、读取表架构以及检查和运行查询的工具：

```python
from langchain_community.agent_toolkits import SQLDatabaseToolkit

toolkit = SQLDatabaseToolkit(db=db, llm=llm)

tools = toolkit.get_tools()

for tool in tools:
    print(f"{tool.name}: {tool.description}\n")
```

**输出:**
```
sql_db_query: This tool's input is a detailed and correct SQL query, and the output is a result from the database. If the query is incorrect, an error message will be returned. If an error is returned, rewrite the query, check the query, and try again. If you encounter an issue with "Unknown column 'xxxx' in 'field list'", use sql_db_schema to query the correct table fields.

sql_db_schema: This tool's input is a comma-separated list of tables, and the output is the schema and sample rows for those tables. Be sure that the tables actually exist by calling sql_db_list_tables first! Example Input: table1, table2, table3

sql_db_list_tables: Input is an empty string, and the output is a comma-separated list of tables in the database.

sql_db_query_checker: Use this tool to double check if your query is correct before executing it. Always use this tool before executing a query with sql_db_query!

```

## 2. 使用预构建的代理

有了这些工具，我们就可以在单行代码中初始化一个预构建的代理。要自定义我们的代理行为，我们编写一个描述性的系统指令。

```python
from langgraph.prebuilt import create_react_agent

system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

You MUST double check your query before executing it. If you get an error while
executing a query, rewrite the query and try again.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the
database.

To start you should ALWAYS look at the tables in the database to see what you
can query. Do NOT skip this step.

Then you should query the schema of the most relevant tables.
""".format(
    dialect=db.dialect,
    top_k=5,
)

agent = create_react_agent(
    llm,
    tools,
    prompt=system_prompt,
)
```

!!! note
    此系统指令包含许多指令，例如始终在其他工具之前或之后运行特定工具。在[下一节](#3-customizing-the-agent)中，我们将通过图的结构来强制执行这些行为，从而为我们提供更大的控制力并允许我们简化指令。

让我们在示例查询上运行此代理并观察其行为：

```python
question = "Which genre on average has the longest tracks?"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
```

**输出:**
```
================================ Human Message =================================

Which genre on average has the longest tracks?
================================== Ai Message ==================================
Tool Calls:
  sql_db_list_tables (call_d8lCgywSroCgpVl558nmXKwA)
 Call ID: call_d8lCgywSroCgpVl558nmXKwA
  Args:
================================= Tool Message =================================
Name: sql_db_list_tables

Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== Ai Message ==================================
Tool Calls:
  sql_db_schema (call_nNf6IIUcwMYLIkE0l6uWkZHe)
 Call ID: call_nNf6IIUcwMYLIkE0l6uWkZHe
  Args:
    table_names: Genre, Track
================================= Tool Message =================================
Name: sql_db_schema


CREATE TABLE "Genre" (
    "GenreId" INTEGER NOT NULL, 
    "Name" NVARCHAR(120), 
    PRIMARY KEY ("GenreId")
)

/*
3 rows from Genre table:
GenreId Name
1   Rock
2   Jazz
3   Metal
*/


CREATE TABLE "Track" (
    "TrackId" INTEGER NOT NULL, 
    "Name" NVARCHAR(200) NOT NULL, 
    "AlbumId" INTEGER, 
    "MediaTypeId" INTEGER NOT NULL, 
    "GenreId" INTEGER, 
    "Composer" NVARCHAR(220), 
    "Milliseconds" INTEGER NOT NULL, 
    "Bytes" INTEGER, 
    "UnitPrice" NUMERIC(10, 2) NOT NULL, 
    PRIMARY KEY ("TrackId"), 
    FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"), 
    FOREIGNKEY("GenreId") REFERENCES "Genre" ("GenreId"), 
    FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
3 rows from Track table:
TrackId Name    AlbumId MediaTypeId GenreId Composer    Milliseconds    Bytes   UnitPrice
1   For Those About To Rock (We Salute You) 1   1   1   Angus Young, Malcolm Young, Brian Johnson   343719  11170334    0.99
2   Balls to the Wall   2   2   1   None    342562  5510424 0.99
3   Fast As a Shark 3   2   1   F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman 230619  3990994 0.99
*/
================================== Ai Message ==================================
Tool Calls:
  sql_db_query_checker (call_urTRmtiGtTxkwHtscec7Fd2K)
 Call ID: call_urTRmtiGtTxkwHtscec7Fd2K
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.Name
ORDER BY AvgMilliseconds DESC
LIMIT 1;
================================= Tool Message =================================
Name: sql_db_query_checker

\`\`\`sql
SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.Name
ORDER BY AvgMilliseconds DESC
LIMIT 1;
\`\`\`
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_RNMqyUEMv0rvy0UxSwrXY2AV)
 Call ID: call_RNMqyUEMv0rvy0UxSwrXY2AV
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgMilliseconds
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.Name
ORDER BY AvgMilliseconds DESC
LIMIT 1;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385)]
================================== Ai Message ==================================

The genre with the longest average track length is "Sci Fi & Fantasy," with an average duration of about 2,911,783 milliseconds (approximately 48.5 minutes) per track.
```

这效果很好：代理正确地列出了表，获取了架构，编写了查询，检查了查询，并运行它来形成最终响应。

!!! tip
    您可以在[LangSmith 跟踪](https://smith.langchain.com/public/bd594960-73e3-474b-b6f2-db039d7c713a/r)中检查上述运行的所有方面，包括执行的步骤、调用的工具、LLM 看到的提示等等。

## 3. 自定义代理

预构建的代理让我们能够快速入门，但在每个步骤中，代理都可以访问所有工具。上面，我们依赖系统指令来约束其行为——例如，我们指示代理始终首先使用“list tables”工具，并在执行查询之前始终运行查询检查器工具。

我们可以通过自定义代理在 LangGraph 中强制执行更高程度的控制。下面，我们实现一个简单的 ReAct 代理设置，为特定的工具调用提供了专用节点。我们将使用与预构建代理相同的[状态](../../concepts/low_level.md#state)。

我们为以下步骤构建了专用节点：

- 列出数据库表
- 调用“get schema”工具
- 生成查询
- 检查查询

将这些步骤放入专用节点使我们能够 (1) 在需要时强制执行工具调用，以及 (2) 自定义与每个步骤相关的提示。

```python
from typing import Literal
from langchain_core.messages import AIMessage
from langchain_core.runnables import RunnableConfig
from langgraph.graph import END, START, MessagesState, StateGraph
from langgraph.prebuilt import ToolNode


get_schema_tool = next(tool for tool in tools if tool.name == "sql_db_schema")
get_schema_node = ToolNode([get_schema_tool], name="get_schema")

run_query_tool = next(tool for tool in tools if tool.name == "sql_db_query")
run_query_node = ToolNode([run_query_tool], name="run_query")


# Example: create a predetermined tool call
def list_tables(state: MessagesState):
    tool_call = {
        "name": "sql_db_list_tables",
        "args": {},
        "id": "abc123",
        "type": "tool_call",
    }
    tool_call_message = AIMessage(content="", tool_calls=[tool_call])

    list_tables_tool = next(tool for tool in tools if tool.name == "sql_db_list_tables")
    tool_message = list_tables_tool.invoke(tool_call)
    response = AIMessage(f"Available tables: {tool_message.content}")

    return {"messages": [tool_call_message, tool_message, response]}


# Example: force a model to create a tool call
def call_get_schema(state: MessagesState):
    # Note that LangChain enforces that all models accept `tool_choice="any"`
    # as well as `tool_choice=<string name of tool>`.
    llm_with_tools = llm.bind_tools([get_schema_tool], tool_choice="any")
    response = llm_with_tools.invoke(state["messages"])

    return {"messages": [response]}


generate_query_system_prompt = """
You are an agent designed to interact with a SQL database.
Given an input question, create a syntactically correct {dialect} query to run,
then look at the results of the query and return the answer. Unless the user
specifies a specific number of examples they wish to obtain, always limit your
query to at most {top_k} results.

You can order the results by a relevant column to return the most interesting
examples in the database. Never query for all the columns from a specific table,
only ask for the relevant columns given the question.

DO NOT make any DML statements (INSERT, UPDATE, DELETE, DROP etc.) to the database.
""".format(
    dialect=db.dialect,
    top_k=5,
)


def generate_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": generate_query_system_prompt,
    }
    # We do not force a tool call here, to allow the model to
    # respond naturally when it obtains the solution.
    llm_with_tools = llm.bind_tools([run_query_tool])
    response = llm_with_tools.invoke([system_message] + state["messages"])

    return {"messages": [response]}


check_query_system_prompt = """
You are a SQL expert with a strong attention to detail.
Double check the {dialect} query for common mistakes, including:
- Using NOT IN with NULL values
- Using UNION when UNION ALL should have been used
- Using BETWEEN for exclusive ranges
- Data type mismatch in predicates
- Properly quoting identifiers
- Using the correct number of arguments for functions
- Casting to the correct data type
- Using the proper columns for joins

If there are any of the above mistakes, rewrite the query. If there are no mistakes,
just reproduce the original query.

You will call the appropriate tool to execute the query after running this check.
""".format(dialect=db.dialect)


def check_query(state: MessagesState):
    system_message = {
        "role": "system",
        "content": check_query_system_prompt,
    }

    # Generate an artificial user message to check
    tool_call = state["messages"][-1].tool_calls[0]
    user_message = {"role": "user", "content": tool_call["args"]["query"]}
    llm_with_tools = llm.bind_tools([run_query_tool], tool_choice="any")
    response = llm_with_tools.invoke([system_message, user_message])
    response.id = state["messages"][-1].id

    return {"messages": [response]}
```

最后，我们使用 Graph API 将这些步骤组装成一个工作流。我们定义了一个[条件边](../../concepts/low_level.md#conditional-edges)在查询生成步骤，如果生成了查询则路由到查询检查器，如果 LLM 已经响应了查询则不存在工具调用，则结束。

```python
def should_continue(state: MessagesState) -> Literal[END, "check_query"]:
    messages = state["messages"]
    last_message = messages[-1]
    if not last_message.tool_calls:
        return END
    else:
        return "check_query"


builder = StateGraph(MessagesState)
builder.add_node(list_tables)
builder.add_node(call_get_schema)
builder.add_node(get_schema_node, "get_schema")
builder.add_node(generate_query)
builder.add_node(check_query)
builder.add_node(run_query_node, "run_query")

builder.add_edge(START, "list_tables")
builder.add_edge("list_tables", "call_get_schema")
builder.add_edge("call_get_schema", "get_schema")
builder.add_edge("get_schema", "generate_query")
builder.add_conditional_edges(
    "generate_query",
    should_continue,
)
builder.add_edge("check_query", "run_query")
builder.add_edge("run_query", "generate_query")

agent = builder.compile()
```

我们在下面可视化应用程序：

```python
from IPython.display import Image, display
from langchain_core.runnables.graph import CurveStyle, MermaidDrawMethod, NodeStyles

display(Image(agent.get_graph().draw_mermaid_png()))
```

![Graph](./output.png)

**注意:** 当您运行此代码时，它将生成并显示 SQL 代理图的视觉表示，显示不同节点之间的流程（list_tables → call_get_schema → get_schema → generate_query → check_query → run_query）。

我们现在可以像以前一样调用该图：

```python
question = "Which genre on average has the longest tracks?"

for step in agent.stream(
    {"messages": [{"role": "user", "content": question}]},
    stream_mode="values",
):
    step["messages"][-1].pretty_print()
```

**输出:**
```
================================ Human Message =================================

Which genre on average has the longest tracks?
================================== Ai Message ==================================

Available tables: Album, Artist, Customer, Employee, Genre, Invoice, InvoiceLine, MediaType, Playlist, PlaylistTrack, Track
================================== Ai Message ==================================
Tool Calls:
  sql_db_schema (call_qxKtYiHgf93AiTDin9ez5wFp)
 Call ID: call_qxKtYiHgf93AiTDin9ez5wFp
  Args:
    table_names: Genre,Track
================================= Tool Message =================================
Name: sql_db_schema


CREATE TABLE "Genre" (
    "GenreId" INTEGER NOT NULL, 
    "Name" NVARCHAR(120), 
    PRIMARY KEY ("GenreId")
)

/*
3 rows from Genre table:
GenreId Name
1   Rock
2   Jazz
3   Metal
*/


CREATE TABLE "Track" (
    "TrackId" INTEGER NOT NULL, 
    "Name" NVARCHAR(200) NOT NULL, 
    "AlbumId" INTEGER, 
    "MediaTypeId" INTEGER NOT NULL, 
    "GenreId" INTEGER, 
    "Composer" NVARCHAR(220), 
    "Milliseconds" INTEGER NOT NULL, 
    "Bytes" INTEGER, 
    "UnitPrice" NUMERIC(10, 2) NOT NULL, 
    PRIMARY KEY ("TrackId"), 
    FOREIGN KEY("MediaTypeId") REFERENCES "MediaType" ("MediaTypeId"), 
    FOREIGNKEY("GenreId") REFERENCES "Genre" ("GenreId"), 
    FOREIGN KEY("AlbumId") REFERENCES "Album" ("AlbumId")
)

/*
3 rows from Track table:
TrackId Name    AlbumId MediaTypeId GenreId Composer    Milliseconds    Bytes   UnitPrice
1   For Those About To Rock (We Salute You) 1   1   1   Angus Young, Malcolm Young, Brian Johnson   343719  11170334    0.99
2   Balls to the Wall   2   2   1   None    342562  5510424 0.99
3   Fast As a Shark 3   2   1   F. Baltes, S. Kaufman, U. Dirkscneider & W. Hoffman 230619  3990994 0.99
*/
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_RPN3GABMfb6DTaFTLlwnZxVN)
 Call ID: call_RPN3GABMfb6DTaFTLlwnZxVN
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgTrackLength
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.GenreId
ORDER BY AvgTrackLength DESC
LIMIT 1;
================================== Ai Message ==================================
Tool Calls:
  sql_db_query (call_PR4s8ymiF3ZQLaoZADXtdqcl)
 Call ID: call_PR4s8ymiF3ZQLaoZADXtdqcl
  Args:
    query: SELECT Genre.Name, AVG(Track.Milliseconds) AS AvgTrackLength
FROM Track
JOIN Genre ON Track.GenreId = Genre.GenreId
GROUP BY Genre.GenreId
ORDER BY AvgTrackLength DESC
LIMIT 1;
================================= Tool Message =================================
Name: sql_db_query

[('Sci Fi & Fantasy', 2911783.0384615385)]
================================== Ai Message ==================================

The genre with the longest tracks on average is "Sci Fi & Fantasy," with an average track length of approximately 2,911,783 milliseconds.
```

!!! tip
    请参阅[LangSmith 跟踪](https://smith.langchain.com/public/94b8c9ac-12f7-4692-8706-836a1f30f1ea/r)以了解上述运行情况。

## 后续步骤

请查看[此指南](https://docs.smith.langchain.com/evaluation/how_to_guides/langgraph)，了解如何使用 LangSmith 评估 LangGraph 应用程序，包括此类 SQL 代理。