# 概览

LangGraph 为希望构建强大、灵活的 AI 代理的开发人员而设计。开发人员选择 LangGraph 是因为它能提供：

- **可靠性和可控性。** 通过审核检查和人工审批来指导代理行为。LangGraph 会持久化上下文以支持长时间运行的工作流，确保您的代理始终朝着正确的方向前进。
- **底层和可扩展性。** 使用完全描述性的底层原语构建自定义代理，摆脱限制定制的僵化抽象。设计可扩展的多代理系统，每个代理都可以根据您的用例扮演特定的角色。
- **一流的流式传输支持。** 通过逐个 token 的流式传输和中间步骤的流式传输，LangGraph 让用户能够实时清晰地了解代理的推理和操作。

## 学习 LangGraph 基础知识

为了熟悉 LangGraph 的关键概念和功能，请完成以下 LangGraph 基础知识系列教程：

1. [构建一个基础聊天机器人](../tutorials/get-started/1-build-basic-chatbot.md)
2. [添加工具](../tutorials/get-started/2-add-tools.md)
3. [添加记忆](../tutorials/get-started/3-add-memory.md)
4. [添加人工干预控制](../tutorials/get-started/4-human-in-the-loop.md)
5. [自定义状态](../tutorials/get-started/5-customize-state.md)
6. [时间回溯](../tutorials/get-started/6-time-travel.md)

通过完成这一系列教程，您将使用 LangGraph 构建一个支持聊天机器人，该机器人能够：

* ✅ 通过网络搜索来**回答常见问题**
* ✅ 在调用之间**维护对话状态**
* ✅ 将**复杂查询路由**给人工审核
* ✅ 使用**自定义状态**来控制其行为
* ✅ **回溯和探索**其他对话路径