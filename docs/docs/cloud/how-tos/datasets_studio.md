# 将节点添加到数据集

本指南将展示如何从线程日志中的节点将示例添加到 [LangSmith 数据集](https://docs.smith.langchain.com/evaluation/how_to_guides#dataset-management)。这对于评估代理的单个步骤非常有用。

1. 选择一个线程。
2. 点击 `Add to Dataset` 按钮。
3. 选择您想添加输入/输出到数据集的节点。
4. 对于每个选定的节点，选择目标数据集来创建示例。默认情况下，将为特定的助手和节点选择数据集。如果此数据集尚不存在，它将被创建。
5. 在将输入/输出添加到数据集之前，按需编辑示例。
6. 选择页面底部的“Add to dataset”将所有选定的节点添加到它们各自的数据集中。

有关如何评估中间步骤的更多详细信息，请参阅 [评估中间步骤](https://docs.smith.langchain.com/evaluation/how_to_guides/langgraph#evaluating-intermediate-steps)。