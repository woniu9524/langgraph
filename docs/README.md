# LangGraph 文档

有关贡献文档的更多信息，请参阅 [贡献指南](../CONTRIBUTING.md)。

## 结构

主要文档位于 `docs/` 目录中。该目录包含主文档的源文件以及 API 参考文档的构建过程。

### 主文档

主文档文件位于 `docs/docs/`，并使用 Markdown 格式编写。该站点使用 [**MkDocs**](https://www.mkdocs.org/) 和 [Material 主题](https://squidfunk.github.io/mkdocs-material/)，并包含：

- **概念**: LangGraph 核心概念和解释
- **教程**: 分步学习指南
- **操作指南 (How-tos)**: 针对特定用例的任务型指南
- **示例**: 实际应用和用例
- **Jupyter Notebooks**: 自动转换为 Markdown 的交互式教程

### API 参考

API 参考文档定义在 `docs/docs/reference/` 中。每个 `.md` 文件都概述了用于构建每个页面的“模板”。使用 **mkdocstrings** 插件，参考内容会从代码库中的 docstrings （文档字符串）自动生成。生成后，内容会被插入到相应的 Markdown 文件中，通过手动指令来指定要记录哪些类和/或函数：

```markdown
::: langgraph.graph.state.StateGraph
    options:
      show_if_no_docstring: true
      show_root_heading: true
      show_root_full_path: false
      members:
        - add_node
        - add_edge
        - add_conditional_edges
        - add_sequence
        - compile
```

## 构建过程

文档构建遵循以下步骤：

1. **内容处理:**
   - `_scripts/notebook_hooks.py` - 主要处理流程，其中：
     - 使用 `notebook_convert.py` 将操作指南/教程 Jupyter Notebooks 转换为 Markdown
     - 使用 `generate_api_reference_links.py` 添加自动 API 参考链接到代码块
     - 处理 Python/JS 版本的条件渲染
     - 处理高亮注释和自定义语法

2. **API 参考生成:**
   - **mkdocstrings** 插件从 Python 源代码提取 docstrings
   - 参考页面 (`/docs/docs/*`) 中的手动 `::: module.Class` 指令指定要记录内容
   - 文档和 API 之间的交叉引用会自动生成

3. **站点生成:**
   - **MkDocs** 处理所有 Markdown 文件并生成静态 HTML
   - 自定义钩子处理重定向并注入额外功能

4. **部署:**
   - 站点通过 Vercel 部署
   - `make build-docs` 生成生产构建（也可用于本地测试）
   - 自动重定向处理不同版本之间的 URL 更改

### 本地开发

本地开发请使用 Makefile 目标：

```bash
# 使用热重载在本地提供文档服务
make serve-docs

# 清理构建以进行生产测试
make build-docs

# 使用清理后的构建提供服务
make serve-clean-docs
```

`serve-docs` 命令：

- 监视源文件变化
- 包含脏构建以加快迭代速度
- 在 [http://127.0.0.1:8000/langgraph/](http://127.0.0.1:8000/langgraph/) 上提供服务

## 标准

**Docstring 格式:**
API 参考使用 **Google 风格的 docstrings** 并带 Markdown 标记。`mkdocstrings` 插件会处理这些内容以生成文档。

**必需格式:**

```python
def example_function(param1: str, param2: int = 5) -> bool:
    """函数的简要描述。

    更长的描述可以在这里写。使用 Markdown 语法进行
    富文本格式，如 **粗体** 和 *斜体*。

    Args:
        param1: 第一个参数的描述。
        param2: 第二个参数的描述，包含默认值。

    Returns:
        返回值描述。

    Raises:
        ValueError: 当 param1 为空时。
        TypeError: 当 param2 不是整数时。

    !!! warning
        此函数是实验性的，可能会发生更改。

    !!! version-added "Added in version 0.2.0"
    """
```

**特殊标记:** 

- **MkDocs 提示框 (admonitions)**: `!!! warning`、`!!! note`、`!!! version-added`
- **代码块**: 标准 Markdown ``` 语法
- **交叉引用**: 通过 `generate_api_reference_links.py` 自动链接

## 执行 Notebooks

如果您想自动执行所有 Notebook，以模拟“运行 Notebooks”（Run notebooks）GitHub Action，您可以运行：

```bash
python _scripts/prepare_notebooks_for_ci.py
./_scripts/execute_notebooks.sh
```

**注意**: 如果您想在不运行 Notebook 中的 `%pip install` 单元格的情况下运行 Notebooks，可以运行：

```bash
python _scripts/prepare_notebooks_for_ci.py --comment-install-cells
./_scripts/execute_notebooks.sh
```

`prepare_notebooks_for_ci.py` 脚本将为 Notebook 中的每个单元格添加 VCR 磁带上下文管理器（VCR cassette context manager），以便：

- 首次运行 Notebook 时，带有网络请求的单元格将被记录到 VCR 磁带文件中
- 之后再次运行 Notebook 时，带有网络请求的单元格将从磁带中回放

## 添加新的 Notebooks

如果您正在添加一个包含 API 请求的 Notebook，**强烈建议**记录网络请求，以便之后可以回放。如果未执行此操作，Notebook 运行器每次运行 Notebook 时都会发出 API 请求，这可能会产生高额费用且速度缓慢。

要记录网络请求，请确保首先运行 `prepare_notebooks_for_ci.py` 脚本。

然后，运行

```bash
jupyter execute <path_to_notebook>
```

Notebook 执行完成后，您应该会在 `cassettes` 目录中看到新记录的 VCR 磁带，并可以弃置更新后的 Notebook。

## 更新现有 Notebooks

如果您正在更新现有的 Notebook，请确保删除 `cassettes` 目录中该 Notebook 的任何现有磁带（每个磁带都以 Notebook 名称为前缀），然后按照“添加新的 Notebooks”部分中的步骤进行操作。

要删除 Notebook 的磁带，您可以运行：

```bash
rm cassettes/<notebook_name>*
```