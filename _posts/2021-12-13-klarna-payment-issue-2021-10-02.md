---
layout: single
title:  "klarna payment issue in europe, 2021-10-2"
---
# klarnas payment issue in europe, 2021-10-2

## 案例原始资料
- [klarnas payment issue in europe, 2021-10-2](https://www.klarna.com/us/blog/detailed-incident-report-disturbances-to-klarnas-payment-methods-in-europe-on-october-2-2021/)

## 案例回顾
- Our applications run on top of what we call a runtime platform. One of the jobs of the runtime platform is to deliver these technical logs to a system which processes technical logs, called the “log-forwarder”.
- To summarize what happened, in the Spring of 2021 we made changes to improve how we release applications on the runtime platform. As part of this, we inadvertently introduced a change in how technical logs are delivered to the log-forwarder
- This change did not cause any unexpected behavior for four months until the log-forwarder experienced increased response times due to an unrelated configuration change. 

1. 2021-04-26 09:36: We merged runtime platform improvements, which included an inadvertent change to how technical logs were delivered to the log-forwarder.
1. 2021-07-21 14:21: The runtime platform improvements were gradually adopted across the organization.
1. 2021-09-07 16:15: Log-forwarder configuration changed from `drop_newest` to `block`.
1. 2021-10-01 12:55: Log-forwarder configuration changed to enable us to measure the size and cardinality of the technical logs
1. 2021-10-02 13:41: An alert on log-forwarder was triggered, which sent an automatic page to on-call.
1. 2021-10-02 14:10: We deployed mitigating changes for log-forwarder.
1. 2021-10-02 14:10: All purchase flows were fully recovered and transacted at full capacity.

Since the log-forwarder does not get overloaded by CPU or memory, the standard metrics for resource starvation do not trigger alerts for this specific application, requiring us to implement custom alerting logic. This incident demonstrated that the logic we implemented was not complete and thus made our alerting insufficient. Upon further investigation we have corrected the specific alert as well as identified other metrics which will now alert if we see similar symptoms in the future.（黄金指标应该被关注）

Improving our procedures for responding to large scale failures where the root cause is not apparent.
