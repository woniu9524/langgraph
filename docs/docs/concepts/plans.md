---
search:
  boost: 2
---

# LangGraph 平台计划


## 概述
LangGraph Platform 是一种用于在生产环境中部署代理式应用程序的解决方案。
它提供了三种不同的使用计划。

- **Developer**: 所有 [LangSmith](https://smith.langchain.com/) 用户都可以使用此计划。只需创建 LangSmith 账户即可获得此计划。该计划支持 [本地部署](./deployment_options.md#free-deployment) 选项。
- **Plus**: 所有拥有 [Plus 账户](https://docs.smith.langchain.com/administration/pricing) 的 [LangSmith](https://smith.langchain.com/) 用户都可以使用此计划。只需将您的 LangSmith 账户升级到 Plus 计划即可获得此计划。该计划支持 [云部署](./deployment_options.md#cloud-saas) 选项。
- **Enterprise**: 此计划独立于 LangSmith 计划。您可以通过[联系我们的销售团队](https://www.langchain.com/contact-sales) 来获得此计划。该计划支持所有[部署选项](./deployment_options.md)。


## 计划详情

|                                                                  | Developer                                   | Plus                                                  | Enterprise                                          |
|------------------------------------------------------------------|---------------------------------------------|-------------------------------------------------------|-----------------------------------------------------|
| 部署选项                                               | 本地                          | 云 SaaS                                         | <ul><li>云 SaaS</li><li>自托管数据平面</li><li>自托管控制平面</li><li>独立容器</li></ul> |
| 用量                                                            | 免费 | 查看 [定价](https://www.langchain.com/langgraph-platform-pricing) | 定制                                              |
| 用于检索和更新状态和对话历史的 API                                               | ✅                                           | ✅                                                     | ✅                                                   |
| 用于检索和更新长期记忆的 API                                               | ✅                                           | ✅                                                     | ✅                                                   |
| 水平可伸缩的任务队列和服务器                                               | ✅                                           | ✅                                                     | ✅                                                   |
| 输出和中间步骤的实时流式传输                                               | ✅                                           | ✅                                                     | ✅                                                   |
| 助手 API（LangGraph 应用的可配置模板）                                       | ✅                                           | ✅                                                     | ✅                                                   |
| Cron 计划                                                        | --                                          | ✅                                                     | ✅                                                   |
| 用于原型设计的 LangGraph Studio                                 | 	✅                                         | ✅                                                    | ✅                                                  |
| 用于调用 LangGraph API 的身份验证和授权                                       | --                                          | 即将推出！                                          | 即将推出！                                        |
| 智能缓存以减少到 LLM API 的流量                                 | --                                          | 即将推出！                                          | 即将推出！                                        |
| 用于状态的发布/订阅 API                                            | --                                          | 即将推出！                                          | 即将推出！                                        |
| 计划优先级                                                   | --                                          | 即将推出！                                          | 即将推出！                                        |

有关定价信息，请参阅[LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)。

## 相关

更多信息，请参阅：

* [部署选项概念指南](./deployment_options.md)
* [LangGraph 平台定价](https://www.langchain.com/langgraph-platform-pricing)
* [LangSmith 计划](https://docs.smith.langchain.com/administration/pricing)