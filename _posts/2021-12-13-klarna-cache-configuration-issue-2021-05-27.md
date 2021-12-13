## Config errors
>[incident-report-incorrect-cache-configuration-leading-to-klarna-app-exposing-personal-information](https://www.klarna.com/us/blog/detailed-incident-report-incorrect-cache-configuration-leading-to-klarna-app-exposing-personal-information/)

- 问题发现途径：终端用户反馈
- 故障发现时间：9分钟
- 故障升级时间：第 11分钟
- 回滚配置文件：第29分钟
- 故障定位时间：31分钟
- 故障完全恢复：第46分钟

改进措施

一些背后的东西：

- Klarna，在故障处理的过程中，有分级和升级机制。
- 依靠终端用户反馈发现问题，而非监控。
- 有故障应急处理团队来全权处理，但是临时召集。assembled an incident management team
- 首先检查变更事件，但是第一次怀疑没有命中。They immediately started to evaluate all recent changes made to our production environment. One of the first updates identified as a potential cause was a change to our Content Delivery Network (CDN) configuration.
- 不同的组三个工程师做peer review？The change was then reviewed by three other authorized engineers in three separate teams, according to normal processes. None of them spotted the mistake in the configuration update.
- 有自动化测试验证。Automated tests did not cover verifying the cache configuration. The configuration change was an unintended side effect and part of a larger change.
- 10天前已经上线到了测试环境。The change had been deployed into an internal testing environment on May 17,

- [incident-report-disturbances-to-klarnas-payment-methods-in-europe-on-october-2-2021](https://www.klarna.com/us/blog/detailed-incident-report-disturbances-to-klarnas-payment-methods-in-europe-on-october-2-2021/)