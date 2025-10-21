# AGENTS 指令

此仓库是一个单体仓库 (monorepo)。每个库都位于 `libs/` 目录下的子目录中。

当你修改任何库中的代码时，请在创建 pull request 之前，在该库的目录下运行以下命令：

- `make format` – 运行代码格式化工具
- `make lint` – 运行 linter
- `make test` – 执行测试套件

要运行特定的测试文件或传递额外的 pytest 选项，你可以指定 `TEST` 变量：

```
TEST=path/to/test.py make test
```

其他 pytest 参数也可以放在 `TEST` 变量中。

## 库 (Libraries)

该仓库包含多个 Python 和 JavaScript/TypeScript 库。
以下是高层次的概述：

- **checkpoint** – LangGraph checkpointer 的基础接口。
- **checkpoint-postgres** – checkpoint saver 的 Postgres 实现。
- **checkpoint-sqlite** – checkpoint saver 的 SQLite 实现。
- **cli** – LangGraph 的官方命令行界面。
- **langgraph** – 用于构建有状态、多参与者代理的核心框架。
- **prebuilt** – 用于创建和运行代理及工具的高级 API。
- **sdk-js** – 用于与 LangGraph REST API 交互的 JS/TS SDK。
- **sdk-py** – LangGraph Server API 的 Python SDK。

### 依赖关系图

下图列出了每个生产依赖项的下游库，这些依赖项在其 `pyproject.toml`（或 `package.json`）中声明。

```text
checkpoint
├── checkpoint-postgres
├── checkpoint-sqlite
├── prebuilt
└── langgraph

prebuilt
└── langgraph

sdk-py
├── langgraph
└── cli

sdk-js (standalone)
```

对某个库的更改可能会影响上面显示的所有其依赖项。