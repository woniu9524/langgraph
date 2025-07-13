---
search:
  boost: 2
---

# 回溯时间 ⏱️

在使用基于模型的非确定性系统（例如，由 LLM 驱动的代理）时，详细检查其决策过程会非常有用：

1. 🤔 **理解推理过程**：分析导致成功结果的步骤。
2. 🐞 **调试错误**：确定错误发生的位置和原因。
3. 🔍 **探索替代方案**：测试不同的路径以发现更好的解决方案。

LangGraph 提供[回溯时间功能](../how-tos/human_in_the_loop/time-travel.md)来支持这些用例。具体来说，您可以从先前的检查点恢复执行——可以重放相同状态，也可以修改它来探索替代方案。在所有情况下，恢复过去的执行都会在历史记录中产生一个新的分支。

!!! tip

    有关如何使用回溯时间的信息，请参阅[使用回溯时间](../how-tos/human_in_the_loop/time-travel.md)和[使用 Server API 进行回溯时间](../cloud/how-tos/human_in_the_loop_time_travel.md)。