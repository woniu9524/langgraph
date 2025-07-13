## Cron 作业

在许多情况下，按计划运行助手非常有用。

例如，假设您正在构建一个助手，该助手每天运行并发送当天的报纸摘要电子邮件。您可以使用 cron 作业每天晚上 8:00 运行该助手。

LangGraph 平台支持 cron 作业，它们会按用户定义的计划运行。用户指定计划、助手和一些输入。之后，在指定的时间，服务器将：

- 使用指定的助手创建一个新线程
- 将指定的输入发送到该线程

请注意，每次发送到线程的输入都是相同的。请参阅 [操作指南](../../cloud/how-tos/cron_jobs.md) 以了解如何创建 cron 作业。

LangGraph 平台 API 提供了几个用于创建和管理 cron 作业的端点。请参阅 [API 参考](../../cloud/reference/api/api_ref.html#tag/runscreate/POST/threads/{thread_id}/runs/crons) 以获取更多详细信息。