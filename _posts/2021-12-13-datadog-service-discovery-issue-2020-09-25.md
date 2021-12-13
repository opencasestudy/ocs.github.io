---
layout: single
title:  "service discovery issue of datadog in 2020-09-25"
---
# service discovery issue of datadog in 2020-09-25

## 案例原始资料
- [Datadog, 2020-09-25-infrastructure-connectivity-issue](https://www.datadoghq.com/blog/2020-09-25-infrastructure-connectivity-issue/)

## 案例回顾

> **Why did it happen?**
> 
> In all regions, the Datadog platform is deployed across multiple availability zones and is routinely tested for resilience against the random loss of nodes in all availability zones. This incident was the result of a kind of failure that we had not experienced before.

**OCS<sub>1</sub>:**
* 公有云上，多region部署，每个 region 多 az 部署。
* 针对多az，都会有定期的故障注入，比如随机丢掉一定量的nodes。不过这次故障的原因，是 datadog 技术团队之前未曾遇到的。


>**The failure of a core system**
>
>The incident was caused by the failure of an internal service discovery and dynamic configuration system that the vast majority of Datadog software components rely on. Service discovery is a central directory of all services running at a given time and provides an easy way for services to find where their dependencies are. Dynamic configuration lets us reconfigure services at run-time and is one of the first dependencies that our services query, as they start up.

**OCS:** 
* 由于datadog 内部的服务发现和动态配置管理系统故障引发。

> This system is backed by a highly available, distributed cluster spanning multiple availability zones. It is designed to withstand the loss of one or two of its nodes at any given time without an impact on its ability to:
> 1. Register and deregister services (“service ​foo is now available at this IP”)
> 2. Answer service discovery queries over DNS (“where is service foo​?”)
> 3. Answer configuration queries (“should this option be turned on for this customer’s requests?”)

**OCS:**
* 都是多 az 设计和部署的。服务的注册和注销；服务的DNS解析；配置开关的请求；

> Because service discovery is a core system throughout our entire infrastructure, its failure unfortunately had global effects, made our recovery efforts difficult, and extended the duration of the incident.

**OCS:**
* 因为是最核心的依赖，所以影响面大。但是为什么恢复起来难呢？之前的高可用设计为什么没有生效？

> We traced the origin of the failure to a routine operation by an authorized engineer early that day, on a small-sized cluster, that is itself a dependency of a much larger data intake cluster. The function of this smaller cluster is to measure latency once data has been received by the intake cluster. When a payload is received, the intake cluster asynchronously instructs the latency-measuring one to start and track latency as the payload traverses our processing pipelines.

**OCS:**
* small-sized cluster，类似于一个『哨兵』机制，旁路工作，用于测量数据上报的latency
* 针对这个故障，应急处理团队首先是去回溯最近的变更事件（这个变更事件是被授权过的）

> We designed this interaction to not be in the critical path of incoming data. If the latency-measuring service is down or missing, we get internal notifications to investigate, but the sole customer-facing effect of that failure is to suppress alert notifications downstream. Or so it was until roughly a month before the incident.

**OCS：**
* 首先这个监控测量的小集群和数据接收集群的交互，在设计上，不处于关键路径，如果监测小集群不可用，给终端客户唯一的影响是会抑制告警发送。

> Late August, as part of a migration of that large intake cluster, we applied a set of changes to its configuration, including a faulty one: instead of using a local file for DNS resolution (slow to update reliably but very resilient to failure), the intake cluster started to depend on the local DNS resolver, which itself is a caching proxy to the service discovery system. Once the faulty change was live, there was no visible difference:

**OCS：**
* 一个月前引入的错误变更；
* DNS 故障（DNS服务是服务发现系统的caching proxy）；

> 1. The local DNS resolver did answer more queries but it did not change any intake service level indicator that otherwise would have had us trigger an immediate investigation.
> 2. The local DNS resolver properly cached DNS queries so very few additional DNS queries were received upstream by the service discovery cluster.

**OCS：**
* local DNS resolver，吞吐更大了，缓存也起作用了
* 但是为什么没有引起 SLI 的变化？缓存起作用了，背后的服务发现系统，压力更小了。

> However, there was one crucial exception which we never see during normal operations: NXDOMAIN answers are not cached by the resolver to quickly propagate service deregistration throughout the infrastructure. In other words, a missing entry for the latency-measuring cluster in the service discovery cluster causes the requesting client to keep asking at a rapid clip where to find that service, even if it does not actually need that to ingest data successfully.

**OCS：**
* 对于 NXDOMAIN的解析，local dns resolver 并不会缓存，而是会直接透传给后端的服务发现系统；
* 由于在服务发现系统中，并没有注册 latency-measuring 集群，因此客户端对该服务的解析请求，会引起 NXDOMAIN并透传到服务发现系统。

> With this change in place and its impact missed during the change review, the conditions were set for an unforeseen failure starting with a routine operation a month later.

**OCS：**
* 故障的隐患触发条件，在一个月前已经埋下了。

> Back to our fateful day. When the smaller, latency-measuring cluster was recycled (scaled down and back up again), its nodes were unreachable for enough time for the larger cluster to issue a large volume of DNS requests back to the service discovery system. The volume of DNS requests was multiplied by 10 in 10 short seconds, as shown below:

**OCS：**
‒ 服务发现系统被百倍的流量打穿了，打挂了。 //限流哪里去了？

> This sudden onslaught caused the service discovery cluster to lose its quorum and fail to reliably register and deregister services, and fail to quickly answer DNS requests coming from other parts of our infrastructure. After local DNS caches expired on all nodes, we faced a “thundering herd” on the service discovery cluster, amplifying the issue until its breaking point. The net result was that most of our services could neither reliably find their dependencies nor load their runtime configuration at startup time, thus causing repeated errors until:

**OCS：**
* 最惨的时刻，在大部分主机上的 DNS cache expired的时刻。服务发现系统彻底不能用了。

> 1. Their dependencies were statically available via a local file,
> 2. Their runtime configuration parameters were statically available via a local file, or
> 3. They could reliably depend on the service discovery cluster again.

**OCS：**
* 如何缓解呢，把动态服务发现，替换成静态配置文件。。。
* 直到服务发现系统恢复正常工作。


> **The impact on the web tier**
> 
> The web tier, which terminates all interactive requests from our users, was visibly affected by the incident. It was intermittently available, with error rates in the 60-90% range throughout, as shown below. In practice pages often errored out, or dashboards successfully refreshed only 10% to 40% of the time.

**OCS：**
* web 服务，基本不能用，错误率90%，持续了近乎小半天。

> **Internal response and recovery**
> 
> A few minutes after the intake cluster started to show intermittent failures and roughly 20 minutes before we publicly declared an incident, teams triggered an internal one. It brought together on-call engineers for each of the affected services into a virtual war room and an incident commander to coordinate the overall response.

**OCS：**
* 故障发生20分钟后，对外披露通报此次incident
* internal incident team：包括各个受影响服务的 on-call 工程师，成立虚拟的war room，设置了 incident comm来协调。
* question：故障是如何发现的？是终端用户反馈吗？

> With service discovery and dynamic configuration down, a lot of the tools we have to mitigate failures became unavailable. We could no longer quickly alter the configuration, nor shed load to another part of the infrastructure. Bringing more capacity online across the board had virtually no effect until service discovery was fixed. Adding more capacity to the service discovery cluster proved ineffective, as distributed consensus-based systems require a quorum of nodes to agree on the state of the world. In this case each node of the service discovery cluster was already too loaded to properly admit extra ones and spread the load.

**OCS：**
* 和facebook上次的dns 故障类似，因为服务发现系统故障，内部的很多用于故障调查和止损的工具都不能用了，比如：没有办法快速修改配置，没有办法快速切换流量。
* 临时扩容增加容量，也不好使（如果没有限流，扩容于事无补）
* 给服务发现系统扩容，也不好使（因为共识算法需要大多数节点能够工作，这时候已有节点已经无法工作了）。

> While the service discovery team was working to stabilize the cluster by cutting it off from all its clients and controlling re-admission, all other teams temporarily eliminated the dependency on service discovery and dynamic configuration. This turned out to be an iterative process for which we did not have ready-made, break-the-glass automation. The web tier was affected for a number of hours for the reasons mentioned above. Other backend services were restored before that point because they are by design isolated from the web tier.

**OCS：**
* 服务发现系统的工程师团队，通过把服务发现系统的请求全拒绝掉，再逐步的放开；
* 其他服务的工程师，把对服务发现系统的依赖给降级掉（如前文提到的，修改为静态文件。。）；
* 后端服务先于 web 服务恢复，web服务受影响数个小时；

> **External response**
> 
> Our external response followed our usual playbook: update the status page when the issue is systemic and post updates every 30 minutes until resolution. In hindsight we were not as effective as we should have been to communicate clearly and unequivocally the impact of the incident and the steps we were taking.

**OCS：**
* 对外的故障应急响应，有 playbook（更新status page，每隔30分钟更新status page，直到问题恢复）；
* 从时候诸葛亮的角度来看，故障处理的步骤和影响，整个对外的沟通并不是那么有效，那么明确；（行业痛点：过程的确定性和透明性）；

> In prior incidents we were able to get a sense of an ETA relatively quickly because we could make quantitative predictions on how fast a patch would be deployed or how soon a backlog of incoming data could be absorbed. In the best cases we were able to mitigate the internal incident before it could become customer-facing.
> 
> In this case, with so many services impacted, we scaled our internal response accordingly but did not do the same with our external response and our communication. We focused too much on the public status page and did not have a dedicated role in incident response to make sure our customers received timely updates if they were not watching our status page.

**OCS：**
* ETA 很重要，但是很难（行业痛点）；


> **How do we avoid it in the future?**
> 
> Post-incident, all engineering teams have been involved in forensic investigations in order to understand in depth what happened and summarize all the findings in a copious collection of internal postmortems. Here is the gist of what we are prioritizing now to avoid this type of failure in the future, with work already underway and continuing into Q4 ’20 and beyond.

**OCS：**
* 改进项，持续一个Q

> 1. Further decouple the control plane and the data plane

**OCS：**
* 数据面和控制面分离

> The unfortunate irony of this incident is that there would have been little to no impact if the service discovery and configuration system had been frozen so as to always return the same answers when queried. Instead, by favoring fast, convenient configuration updates and flexible service discovery we inadvertently coupled systems together in subtle ways.
> 
> We had already started to remove that coupling in the following ways, and we will double-down:
> 
> ‒ We have split service discovery and dynamic configuration into separate services.
> ‒ We are building additional layers of cache to make DNS queries for the purpose of service discovery resilient to a prolonged loss of the core service discovery system.

**OCS：**
* 缓存层级还不够多吗？还加？

> ‒ We are hardening all components to make them resilient to a prolonged loss of service discovery and dynamic configuration. When they are unavailable, services that process incoming data or answer interactive queries should fail “closed” and continue to work without degradation.

**OCS：**
* 这是一个关键改进，特别是针对服务发现或者连接配置等信息，如果中心服务端检索不到，那么请用本地的缓存。

> ‒ We will regularly test the types of failures we experienced.

**OCS：**
* 故障注入测试，只能发现过往出现过的case，是一种回归测试。

> Improve the resilience of service discovery
> The need to register and deregister services won’t go away but we must support it with a system that is not a single point of failure (be it distributed or not).

**OCS：**
* 分布式并不等价于就是高可用。分布式本身就是一种工程实现，有他的工作前提。还是要有兜底措施。

> **Improve the resilience of the web tier**
> 
> Because the web tier sits in the middle of all queries made by our customers, it must be among the last systems to fail. This means reducing the number of hard dependencies to the absolute minimum with regular tests to make sure pages still load if soft dependencies fail downstream.

**OCS：**
* 不太懂。

> **Improve our external response**
> 
> It starts with having a dedicated role with clear processes to disseminate updates about high visibility incidents throughout. It also includes having clearer communication on the impact of an incident as it develops.

**OCS：**
* 需要有工具支撑。

> Provide a clear playbook in case of regional failure
> Regardless of how much resilience we build into a Datadog instance running in a single region, there will remain a risk that the region becomes unavailable for one reason or another. We are committed to providing options for our customers to choose for contingency.

**OCS：**
* 针对region级别的故障，针对dd来说，其实可选方案不多。因为dd的数据就是按照region 分区的。


> **In closing**
> 
> This incident has been a frustrating experience for our customers and a humbling moment for all Datadog teams. We are keenly aware of our responsibility as your partner. You trust us and our platform to be your eyes and ears and we are sorry for not living up to it on that day. We are committed to learning from this experience, and to delivering meaningful improvements to our service and our communication.

**OCS：**
* 不出故障是不可能的，做好兜底。

## top3的建议

## top3的借鉴

## 给datadog推荐解决方案

## OCS 点评

## 数据统计