---
search:
  boost: 2
---

# LangGraph 平台计划

## 概述
LangGraph Platform 是一个用于在生产环境中部署代理应用程序的解决方案。
它提供三种不同的使用计划。

- **Developer**: 所有 [LangSmith](https://smith.langchain.com/) 用户均可使用此计划。只需创建一个 LangSmith 账户即可注册此计划。这将使您能够使用 [独立容器 (Lite)](./deployment_options.md) 部署选项。
- **Plus**: 所有拥有 [Plus 账户](https://docs.smith.langchain.com/administration/pricing) 的 [LangSmith](https://smith.langchain.com/) 用户均可使用此计划。只需将您的 LangSmith 账户升级到 Plus 计划即可注册此计划。这将使您能够使用 [云](./deployment_options.md#cloud-saas) 部署选项。
- **Enterprise**: 此计划与 LangSmith 计划是分开的。您可以通过联系 sales@langchain.dev 进行注册。这将使您能够使用所有 [部署选项](./deployment_options.md)。

## 计划详情

|                                                                  | Developer                                   | Plus                                                  | Enterprise                                          |
|------------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------|-----------------------------------------------------|
| Deployment Options                                               | 独立容器 (Lite)                          | 云 SaaS                                         | <ul><li>云 SaaS</li><li>自托管数据平面</li><li>自托管控制平面</li><li>独立容器 (企业版)</li></ul> |
| Usage                                                            | 免费，每年限制为 100 万 [节点执行](../concepts/faq.md#what-does-nodes-executed-mean-for-langgraph-platform-usage) | 请参阅 [定价](https://www.langchain.com/langgraph-platform-pricing) | 自定义                                              |
| 用于检索和更新状态及会话历史的 API | ✅                                           | ✅                                                     | ✅                                                   |
| 用于检索和更新长期记忆的 API | ✅                                           | ✅                                                     | ✅                                                   |
| 水平可扩展的任务队列和服务器                    | ✅                                           | ✅                                                     | ✅                                                   |
| 输出和中间步骤的实时流式传输            | ✅                                           | ✅                                                     | ✅                                                   |
| Assistants API (LangGraph 应用的可配置模板)       | ✅                                           | ✅                                                     | ✅                                                   |
| Cron 调度                                                        | --                                          | ✅                                                     | ✅                                                   |
| 用于原型设计的 LangGraph Studio                                | 	✅                                         | ✅                                                    | ✅                                                  |
| 用于调用 LangGraph API 的身份验证和授权        | --                                          | 即将推出！                                          | 即将推出！                                        |
| 智能缓存，减少 LLM API 流量                  | --                                          | 即将推出！                                          | 即将推出！                                        |
| 用于状态的发布/订阅 API                    | --                                          | 即将推出！                                          | 即将推出！                                        |
| 调度优先级                                         | --                                          | 即将推出！                                          | 即将推出！                                        |

有关定价信息，请参阅 [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)。

## 相关

有关更多信息，请参阅：

* [部署选项概念指南](./deployment_options.md)
* [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)
* [LangSmith 计划](https://docs.smith.langchain.com/administration/pricing)