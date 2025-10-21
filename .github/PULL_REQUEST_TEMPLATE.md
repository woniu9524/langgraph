感谢您为 LangGraph 做出贡献！请按照以下步骤将您的拉取请求（Pull Request）标记为“准备好审查”。**如果这些步骤中的任何一项未完成，您的 PR 将不会被考虑进行审查。**

- [ ] **PR 标题**: 遵循格式：`{类型}({范围}): {描述}`
  - 示例：
    - `feat(core): add multi-tenant support`
    - `fix(cli): resolve flag parsing error`
    - `docs(openai): update API usage examples`
  - 允许的 `{类型}` 值：
    - `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`, `release`
  - 允许的 `{范围}` 值（可选）：
    - `langgraph`, `docs`, `cli`, `checkpoint`, `checkpoint-postgres`, `checkpoint-sqlite`, `prebuilt`, `scheduler-kafka`, `sdk-py`
  - 编写完标题后，请删除此清单项；请勿将其包含在 PR 中。

- [ ] **PR 消息**: ***删除整个清单*** 并替换为：
  - **Description:** 对更改的描述。如果适用，请包含一个[关闭关键字](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/linking-a-pull-request-to-an-issue#linking-a-pull-request-to-an-issue-using-a-keyword)。
  - **Issue:** 如果适用，它修复的问题 #
  - **Dependencies:** 此更改所需的任何依赖项
  - **Twitter handle:** 如果您的 PR 被宣布，并且您希望被提及，我们将很乐意为您宣传！

- [ ] **添加测试和文档**: 如果您正在添加新的集成，则必须包括：
  1. 集成的测试，最好是那些不依赖于网络访问的单元测试；
  2. 一个展示其用法的示例 Notebook。它应放置在 `docs/docs/integrations` 目录中。

- [ ] **格式化和测试**: 从您修改的包的根目录运行 `make format`、`make lint` 和 `make test`。除非这三个命令在 CI 中通过，否则我们不会考虑您的 PR。有关更多信息，请参阅[贡献指南](https://github.com/langchain-ai/langgraph/blob/main/CONTRIBUTING.md)。

其他指南：

- 确保可选的依赖项在函数内部导入。
- 请勿将依赖项添加到 `pyproject.toml` 文件（即使是可选的），除非它们是单元测试的**必需**项。
- 大多数 PR 不应涉及超过一个包。
- 更改应向后兼容。