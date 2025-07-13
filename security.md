# 安全策略

## 举报开源漏洞

LangChain 与 [huntr by Protect AI](https://huntr.com/) 合作，为我们的开源项目提供漏洞赏金计划。

请访问以下链接举报与 LangChain 开源项目相关的安全漏洞：

[https://huntr.com/bounties/disclose/](https://huntr.com/bounties/disclose/?target=https%3A%2F%2Fgithub.com%2Flangchain-ai%2Flangchain&validSearch=true)

在举报漏洞之前，请审阅：

1) 下方的范围之内和范围之外的目标。
2) [langchain-ai/langchain](https://python.langchain.com/docs/contributing/repo_structure) 单体仓库结构。
3) LangChain [安全指南](https://python.langchain.com/docs/security)，以了解我们如何区分安全漏洞与开发者责任。

### 范围之内目标

以下软件包和仓库符合漏洞赏金的资格：

- langchain-core
- langchain (参见例外)
- langchain-community (参见例外)
- langgraph
- langserve

### 范围之外目标

huntr 定义的所有范围之外目标以及：

- **langchain-experimental**：此仓库包含实验性代码，不符合漏洞赏金资格。向其举报的漏洞报告将被标记为“有趣”或“浪费时间”，并在不附带赏金的情况下发布。
- **tools**：langchain 或 langchain-community 中的工具不符合漏洞赏金资格。这包括以下目录：
  - langchain/tools
  - langchain-community/tools
  - 请查阅我们的[安全指南](https://python.langchain.com/docs/security)了解更多详情，但一般来说，工具会与现实世界进行交互。开发者应了解其代码的安全影响，并对其工具的安全性负责。
- 标有安全声明的代码。这将根据具体情况决定，但可能不符合赏金资格，因为代码已包含应遵循的指南，以确保应用程序安全。
- 任何与 LangSmith 相关的仓库或 API，请参阅下文。

## 举报 LangSmith 漏洞

请通过电子邮件 `security@langchain.dev` 举报与 LangSmith 相关的安全漏洞。

- LangSmith 网站：https://smith.langchain.com
- SDK 客户端：https://github.com/langchain-ai/langsmith-sdk

### 其他安全顾虑

对于任何其他安全顾虑，请通过 `security@langchain.dev` 联系我们。